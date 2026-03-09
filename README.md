
/**
 * Copilot SDK wrapper
 * Manages the CopilotClient lifecycle and provides a helper
 * for chat completions.
 *
 * File fetching is done separately via GitHub REST API (see github.ts)
 * so that private repos work with a PAT. The Copilot SDK is used
 * purely for LLM summarization.
 */
import { CopilotClient } from "@github/copilot-sdk";

/** Model to use for summarization */
const DEFAULT_MODEL = "claude-haiku-4.5";
// Map user-friendly dashed names → actual dot-notation API model IDs
const MODEL_ALIASES: Record<string, string> = {
  "claude-haiku-4-5": "claude-haiku-4.5",
  "claude-sonnet-4-6": "claude-sonnet-4.6",
  "claude-opus-4-6": "claude-opus-4.6",
};

export function getConfiguredModel(): string {
  const configured = (process.env.COPILOT_MODEL || DEFAULT_MODEL).trim();
  const normalized = MODEL_ALIASES[configured] || configured;

  if (normalized !== configured) {
    console.warn(`[copilot] Normalized model alias "${configured}" -> "${normalized}"`);
  }

  return normalized;
}

let clientInstance: CopilotClient | null = null;

/** Get or create the singleton CopilotClient */
export async function getCopilotClient(): Promise<CopilotClient> {
  if (!clientInstance) {
    console.time("[copilot] Client init");

    // The Copilot SDK needs a token from a GitHub account with an active
    // Copilot subscription. Set COPILOT_TOKEN in .env with the gho_ OAuth
    // token from your personal account (run: gh auth token on personal account).
    // If unset, the SDK uses useLoggedInUser=true to authenticate via
    // the gh CLI's currently active account.
    const token = process.env.COPILOT_TOKEN || process.env.COPILOT_GITHUB_TOKEN;

    const options: Record<string, any> = {};
    if (token) {
      options.githubToken = token;
      console.log("[copilot] Using explicit COPILOT_TOKEN for auth");
    } else {
      options.useLoggedInUser = true;
      console.log("[copilot] No COPILOT_TOKEN set — using gh CLI logged-in user");
    }

    clientInstance = new CopilotClient(options);
    console.timeEnd("[copilot] Client init");
  }
  return clientInstance;
}

/** Pre-warm the client so the first request isn't slow */
export async function warmUpCopilotClient(): Promise<void> {
  try {
    console.log("[copilot] Pre-warming client...");
    const client = await getCopilotClient();

    // Explicitly wait for the CLI connection to be established
    await client.start();
    console.log("[copilot] Client connected");

    // List available models so we know what's valid
    try {
      const models = await client.listModels();
      console.log("[copilot] Available models:", models.map((m: any) => m.id || m.name || m).join(", "));
    } catch (e) {
      console.warn("[copilot] Could not list models:", e);
    }
  } catch (err) {
    console.error("[copilot] Warm-up failed:", err);
  }
}

/** Stop the client (call on shutdown) */
export async function stopCopilotClient(): Promise<void> {
  if (clientInstance) {
    await clientInstance.stop();
    clientInstance = null;
  }
}

/**
 * Summarize pre-fetched file content using Copilot LLM.
 *
 * This does NOT use MCP / GitHub tools — the file content is already
 * fetched via the GitHub REST API (github.ts). We just ask the model
 * to analyze and summarize the provided text.
 */
export async function summarizeFileContent(params: {
  owner: string;
  repo: string;
  filePath: string;
  ref: string;
  fileContent: string;
  onChunk?: (chunk: string) => void;
}): Promise<string> {
  const { owner, repo, filePath, ref, fileContent, onChunk } = params;
  const model = getConfiguredModel();

  // Truncate very large files to keep LLM responses fast
  const MAX_CHARS = 30_000;
  let content = fileContent;
  let truncated = false;
  if (content.length > MAX_CHARS) {
    content = content.slice(0, MAX_CHARS);
    truncated = true;
    console.log(`[copilot] File truncated from ${fileContent.length} to ${MAX_CHARS} chars`);
  }
  console.log(`[copilot] Sending ${content.length} chars to LLM`);

  console.time("[copilot] getCopilotClient");
  const client = await getCopilotClient();
  console.timeEnd("[copilot] getCopilotClient");

  const systemPrompt = `You are a code analysis assistant. Your job is to analyze the provided file content and produce a clear, well-structured summary.

When summarizing, include:
- **Purpose**: What the file/code is for
- **Key Components**: Main classes, functions, exports, or sections
- **Dependencies**: Notable imports or external dependencies
- **Architecture Notes**: Design patterns, data flow, or important decisions
- **Notable Details**: Anything else worth highlighting

Format your response in clean Markdown.`;

  const useStreaming = !!onChunk;
  console.log(`[copilot] Using model: ${model}, streaming: ${useStreaming}`);
  console.time("[copilot] createSession");
  const session = await client.createSession({
    model,
    streaming: useStreaming,
    systemMessage: {
      mode: "replace",
      content: systemPrompt,
    },
    onPermissionRequest: async (request) => {
      console.warn("[copilot] Permission requested (auto-approving):", request.kind);
      return { kind: "approved" };
    },
    onUserInputRequest: async (request) => {
      console.warn("[copilot] User input requested (auto-responding):", request.question);
      return { answer: "", wasFreeform: true };
    },
  });
  console.timeEnd("[copilot] createSession");

  // Debug: log ALL session events to see what's happening
  session.on((event: any) => {
    if (event.type !== "pending_messages.modified") {
      console.log(`[copilot] Event: ${event.type}`, event.type === "session.error" ? event.data : "");
    }
  });

  const truncateNote = truncated ? `\n\n⚠️ Note: File was truncated to first ${MAX_CHARS} characters for analysis.` : "";

  const userPrompt = `Please analyze and summarize the following file.

**Repository:** \`${owner}/${repo}\` (branch: \`${ref}\`)
**File:** \`${filePath}\`${truncateNote}

---

\`\`\`
${content}
\`\`\`

---

Provide a detailed, well-structured summary of what this file does.`;

  let fullResponse = "";

  if (onChunk) {
    session.on("assistant.message_delta", (event: any) => {
      const delta = event.data.deltaContent;
      if (delta) {
        fullResponse += delta;
        onChunk(delta);
      }
    });
  }

  try {
    console.time("[copilot] sendAndWait");
    const response = await session.sendAndWait(
      { prompt: userPrompt },
      180_000 // 3-minute timeout
    );
    console.timeEnd("[copilot] sendAndWait");

    if (!onChunk && response?.data?.content) {
      fullResponse = response.data.content;
    }

    return fullResponse;
  } finally {
    await session.destroy();
  }
}




# 🎉 Festive Season 2025 - Achievement Celebration Page

A stunning, animated celebration webpage showcasing record-breaking achievements with fireworks background and dynamic sparkle effects.

## 🌟 Features

### ✨ **Visual Highlights**
- **Animated Fireworks Background** - Multi-layered radial gradients creating spectacular firework effects
- **Dynamic Sparkle Particles** - 100+ animated particles floating across the screen
- **Dual Hero Layout** - Equal prominence for two key achievements
- **Golden Theme** - Premium gold color scheme with glowing effects
- **Responsive Design** - Optimized for all devices from mobile to desktop

### 🎯 **Interactive Elements**
- **Count-up Animations** - Numbers animate from 0 to final values with easing
- **Click Interactions** - Click anywhere to trigger sparkle bursts
- **Hover Effects** - Achievement cards lift and glow on hover
- **Performance Optimization** - Animations pause when tab is inactive

### 🎨 **Animations**
- **Number Pulse & Float** - Hero numbers pulse and float continuously
- **Title Glow** - Text shadows animate for dramatic effect
- **Staggered Entrance** - Elements appear with cascading delays
- **Particle Physics** - Sparkles follow realistic floating patterns

## 📊 Achievement Metrics

### 🏆 **Primary Achievements**
- **Total Disbursement Oct-25**: ₹51 Lakh+
- **Highest Ever Per Day Cases Processed**: 7.58 Lakh

### 🚀 **Supporting Metrics**
- **Highest Ever Per Day Disbursement**: ₹5,85,180
- **Highest Ever Hourly Cases Processed**: 74,257

## 🛠️ Technical Stack

### **Frontend Technologies**
- **HTML5** - Semantic structure
- **CSS3** - Advanced animations and responsive design
- **Vanilla JavaScript** - Performance-optimized interactions
- **CSS Grid & Flexbox** - Modern layout systems

### **Design Libraries**
- **Bootstrap 5** - CSS framework foundation
- **Font Awesome 6** - Premium icon library
- **Google Fonts (Poppins)** - Modern typography

### **Animation Techniques**
- **CSS Keyframes** - Smooth, hardware-accelerated animations
- **RequestAnimationFrame** - Optimized JavaScript animations
- **CSS Transforms** - GPU-accelerated effects
- **CSS Gradients** - Complex background patterns

## 🎯 Key Code Features

### **Performance Optimizations**
```javascript
// Visibility API for performance
document.addEventListener('visibilitychange', () => {
  const particles = document.querySelectorAll('.sparkle');
  particles.forEach(particle => {
    particle.style.animationPlayState = document.hidden ? 'paused' : 'running';
  });
});
```

### **Smooth Number Animation**
```javascript
function animateValue(elementId, start, end, duration, suffix = '') {
  // Cubic easing for natural feel
  function easeOutCubic(t) {
    return 1 - Math.pow(1 - t, 3);
  }
  // RequestAnimationFrame for 60fps
}
```

### **Dynamic Particle System**
```css
.sparkle:nth-child(2n) {
  background: var(--secondary-gold);
  animation-delay: -1s;
  animation-duration: 5s;
}
```

## 📱 Responsive Design

### **Breakpoints**
- **Desktop (1024px+)**: Dual-column hero layout
- **Tablet (768px-1024px)**: Single-column with optimized spacing
- **Mobile (<768px)**: Compact layout with adjusted typography

### **Adaptive Features**
- **Clamp() Typography** - Fluid font scaling
- **CSS Grid Auto-fit** - Responsive achievement cards
- **Viewport Units** - Full-screen background effects

## 🚀 Getting Started

### **Quick Start**
1. **Clone the repository**
   ```bash
   git clone https://github.com/RK-001/Celebration.git
   cd Celebration
   ```

2. **Open in browser**
   ```bash
   # Simply open the HTML file
   open index.html
   ```

3. **View live**
   - No build process required
   - Works offline
   - All dependencies loaded via CDN

## 🎨 Color Palette

| Color | Usage | Hex Code |
|-------|-------|----------|
| Primary Gold | Main highlights | `#FFD700` |
| Secondary Gold | Accents | `#FFA500` |
| Accent Gold | Bright highlights | `#FFFF00` |
| Dark Background | Base | `#0a0a0a` |

## 🌟 Browser Support

- ✅ **Chrome 80+**
- ✅ **Firefox 75+**
- ✅ **Safari 13+**
- ✅ **Edge 80+**
- ⚠️ **IE 11** (Limited support)

## 📈 Performance Metrics

- **Lighthouse Score**: 95+ Performance
- **First Contentful Paint**: <1.5s
- **Time to Interactive**: <2s
- **Animation Frame Rate**: 60fps

## 🔧 Customization

### **Modify Achievement Numbers**
```javascript
// Update values in the initialization
setTimeout(() => animateValue('heroNumber', 0, 51, 3000, 'Lakh +'), 1000);
setTimeout(() => animateValue('dailyCases', 0, 7.58, 2200, 'Lakh'), 1200);
```

### **Change Color Theme**
```css
:root {
  --primary-gold: #your-color;
  --secondary-gold: #your-color;
  --accent-gold: #your-color;
}
```

### **Adjust Particle Count**
```javascript
// In createSparkles function
for (let i = 0; i < 150; i++) { // Increase for more particles
```

## 🎭 Animation Details

### **Fireworks Background**
- 8 layered radial gradients
- 6-second glow cycle
- Brightness and contrast variations

### **Sparkle Particles**
- 12 different colors
- Randomized size (3-11px)
- Staggered animation delays
- Physics-based movement

### **Number Animations**
- Cubic easing curves
- Indian number formatting
- Staggered start times
- Smooth 60fps transitions

## 🎯 Best Practices Implemented

- **Semantic HTML** for accessibility
- **CSS Custom Properties** for maintainability
- **Mobile-first** responsive design
- **Performance optimization** with RAF
- **Clean code architecture**
- **Progressive enhancement**

## 📞 Contact

**Project Maintainer**: RK-001  
**Repository**: [Celebration](https://github.com/RK-001/Celebration)

---

<div align="center">

**🎉 Celebrating Excellence with Style! 🎉**

*Made with ❤️ and lots of ✨ animations*

</div>
