# Showcase Page UI Plan: Math Notation Overlay

A detailed implementation plan for the showcase page layout featuring interactive mathematical formulas alongside the lightning visualization.

---

## 1. Layout Architecture

### 1.1 Overall Structure

```
+------------------------------------------------------------------+
|                                                                  |
|  +------------------------+                                      |
|  |                        |                                      |
|  |     MATH PANEL         |        LIGHTNING CANVAS              |
|  |     (left 2/3)         |        (right 2/3)                   |
|  |                        |                                      |
|  |  [formulas]  [fade]====|========[animation]=================  |
|  |                   -----|-----------------------------------------
|  |              gradient  |        (overlap zone)                |
|  |              fade-out  |                                      |
|  +------------------------+                                      |
|                                                                  |
|                    [Parameter Sliders at Bottom]                 |
|                                                                  |
+------------------------------------------------------------------+
```

**Key dimensions:**
- Canvas: Full viewport, positioned absolutely
- Math Panel: `width: 66vw`, positioned from `left: 0`
- Animation visual center: shifted ~10-15% right of true center
- Overlap zone: center third of viewport (both panel fade and animation visible)

### 1.2 CSS Layout Strategy

Use absolute positioning with z-index layering. The Canvas remains full-screen; the math panel floats above it.

```css
.showcase-page {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: #000;
}

.showcase-canvas {
  position: absolute;
  inset: 0;
  z-index: 1;
}

.math-panel {
  position: absolute;
  top: 0;
  left: 0;
  width: 66vw;
  height: 100vh;
  z-index: 10;
  pointer-events: none; /* Allow clicking through to canvas */

  /* Semi-transparent dark background with gradient fade */
  background: linear-gradient(
    to right,
    rgba(0, 0, 0, 0.5) 0%,
    rgba(0, 0, 0, 0.5) calc(100% - 100px),
    rgba(0, 0, 0, 0) 100%
  );

  padding: 40px 120px 40px 40px; /* Extra right padding for fade zone */
  box-sizing: border-box;
}

.math-panel > * {
  pointer-events: auto; /* Re-enable pointer events for interactive children */
}

.parameter-controls {
  position: absolute;
  bottom: 24px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 20;
}
```

### 1.3 Animation Offset

Shift the lightning visualization slightly right so the main bolt isn't hidden behind the math panel.

**Option A: Camera offset (preferred)**
```tsx
// In Scene.tsx or ShowcasePage.tsx
<Canvas camera={{ position: [1.5, 0, 6], fov: 50, lookAt: [1, 0, 0] }}>
```

**Option B: Scene offset**
```tsx
// Wrap scene content in a group with x-offset
<group position={[1.2, 0, 0]}>
  <LightningController ... />
  <GroundPlane ... />
</group>
```

The bolt should appear roughly 55-60% from the left edge of the viewport when centered in view.

### 1.4 Responsive Behavior

| Breakpoint | Behavior |
|------------|----------|
| >= 1200px | Full side-by-side layout as described |
| 900-1199px | Math panel: 55vw, smaller font sizes |
| < 900px | Stack layout: math panel becomes bottom sheet (40vh), animation takes top 60vh |
| < 600px | Hide math panel, show minimal parameter controls only |

```css
@media (max-width: 1199px) {
  .math-panel { width: 55vw; }
  .formula-display { font-size: 0.9em; }
}

@media (max-width: 899px) {
  .math-panel {
    top: auto;
    bottom: 0;
    width: 100vw;
    height: 45vh;
    background: linear-gradient(
      to top,
      rgba(0, 0, 0, 0.7) 0%,
      rgba(0, 0, 0, 0.7) calc(100% - 60px),
      rgba(0, 0, 0, 0) 100%
    );
    padding: 20px;
    overflow-y: auto;
  }
}

@media (max-width: 599px) {
  .math-panel { display: none; }
}
```

---

## 2. Math Rendering

### 2.1 Library Choice: KaTeX

**Recommendation: KaTeX** over MathJax.

| Factor | KaTeX | MathJax |
|--------|-------|---------|
| Render speed | 100x faster | Slower |
| Bundle size | ~300KB | ~1MB+ |
| React integration | Excellent | Good |
| Feature set | Sufficient | More complete |
| Output quality | Excellent | Excellent |

For the formulas we need (Greek letters, fractions, subscripts, superscripts, summations), KaTeX is more than capable.

**Installation:**
```bash
npm install katex react-katex
npm install -D @types/katex
```

### 2.2 React Integration

Create a `FormulaDisplay` component that wraps KaTeX:

```tsx
// components/FormulaDisplay.tsx
import { InlineMath, BlockMath } from 'react-katex';
import 'katex/dist/katex.min.css';

interface FormulaDisplayProps {
  latex: string;
  block?: boolean;
  label?: string;
  highlightVars?: string[];
  currentValues?: Record<string, number>;
  className?: string;
}

export const FormulaDisplay: React.FC<FormulaDisplayProps> = ({
  latex,
  block = true,
  label,
  highlightVars = [],
  currentValues = {},
  className = '',
}) => {
  // Process latex to add highlighting to specified variables
  let processedLatex = latex;
  highlightVars.forEach(varName => {
    // Wrap variable in \textcolor{cyan}{...}
    const regex = new RegExp(`\\\\${varName}(?![a-zA-Z])`, 'g');
    processedLatex = processedLatex.replace(regex, `\\textcolor{#4fd1c5}{\\${varName}}`);
  });

  const MathComponent = block ? BlockMath : InlineMath;

  return (
    <div className={`formula-display ${className}`}>
      {label && <div className="formula-label">{label}</div>}
      <MathComponent math={processedLatex} />
      {Object.keys(currentValues).length > 0 && (
        <div className="formula-values">
          {Object.entries(currentValues).map(([key, value]) => (
            <span key={key} className="value-chip">
              {key} = {value}
            </span>
          ))}
        </div>
      )}
    </div>
  );
};
```

### 2.3 Styling for Dark Background

Override KaTeX's default styles for dark theme:

```css
/* KaTeX dark theme overrides */
.math-panel .katex {
  color: #e2e8f0; /* Slightly off-white for readability */
  font-size: 1.1em;
}

.math-panel .katex .textcolor {
  /* Highlighted variables */
}

.formula-display {
  margin-bottom: 24px;
  transition: opacity 0.3s ease;
}

.formula-label {
  font-family: 'Inter', system-ui, sans-serif;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.1em;
  color: #64748b;
  margin-bottom: 8px;
}

.formula-values {
  display: flex;
  gap: 12px;
  margin-top: 8px;
  flex-wrap: wrap;
}

.value-chip {
  font-family: 'JetBrains Mono', monospace;
  font-size: 0.8rem;
  color: #94a3b8;
  background: rgba(255, 255, 255, 0.05);
  padding: 2px 8px;
  border-radius: 4px;
}

/* Highlighted state when slider is active */
.formula-display.highlighted {
  background: rgba(79, 209, 197, 0.05);
  border-left: 2px solid #4fd1c5;
  padding-left: 12px;
  margin-left: -14px;
}
```

### 2.4 Formula Definitions

Store formulas as structured data:

```tsx
// config/formulas.ts
export interface Formula {
  id: string;
  label: string;
  latex: string;
  variables: {
    name: string;
    symbol: string;
    description: string;
    linkedParameter?: string; // Links to a slider parameter
  }[];
}

export const FORMULAS: Formula[] = [
  {
    id: 'growth-probability',
    label: 'Growth Probability (DBM)',
    latex: 'P(i) = \\frac{|\\phi_i|^\\eta}{\\sum_j |\\phi_j|^\\eta}',
    variables: [
      { name: 'phi', symbol: '\\phi', description: 'Electric potential', linkedParameter: null },
      { name: 'eta', symbol: '\\eta', description: 'Branching exponent', linkedParameter: 'eta' },
    ],
  },
  {
    id: 'electric-field',
    label: 'Electric Field',
    latex: '\\vec{E} = -\\nabla\\phi \\approx E_0 \\left(1 + \\frac{k}{d + \\epsilon}\\right)',
    variables: [
      { name: 'E0', symbol: 'E_0', description: 'Background field' },
      { name: 'd', symbol: 'd', description: 'Distance to channel' },
    ],
  },
  {
    id: 'branch-probability',
    label: 'Branch Probability',
    latex: 'P_{\\text{branch}} = P_0 \\cdot \\left(\\frac{E}{E_{\\text{ref}}}\\right)^\\beta \\cdot e^{-\\text{depth}/\\lambda}',
    variables: [
      { name: 'P0', symbol: 'P_0', description: 'Base probability', linkedParameter: 'branchProbability' },
      { name: 'beta', symbol: '\\beta', description: 'Field exponent' },
      { name: 'lambda', symbol: '\\lambda', description: 'Depth decay', linkedParameter: 'branchDecay' },
    ],
  },
  {
    id: 'return-stroke',
    label: 'Return Stroke Current (Heidler)',
    latex: 'I(t) = I_0 \\cdot \\frac{(t/\\tau_1)^2}{(t/\\tau_1)^2 + 1} \\cdot e^{-t/\\tau_2}',
    variables: [
      { name: 'I0', symbol: 'I_0', description: 'Peak current' },
      { name: 'tau1', symbol: '\\tau_1', description: 'Rise time', linkedParameter: 'returnStrokeSpeed' },
      { name: 'tau2', symbol: '\\tau_2', description: 'Decay time' },
    ],
  },
  {
    id: 'luminosity-decay',
    label: 'Luminosity Decay',
    latex: 'L(t) = L_0 \\cdot e^{-t/\\tau}',
    variables: [
      { name: 'L0', symbol: 'L_0', description: 'Initial luminosity' },
      { name: 'tau', symbol: '\\tau', description: 'Decay constant', linkedParameter: 'decayRate' },
    ],
  },
];
```

### 2.5 Visual Hierarchy

Display formulas in logical groups with clear visual separation:

```
+----------------------------------+
| STEPPED LEADER PHYSICS           |  <- Section header
|                                  |
| Growth Probability (DBM)         |  <- Formula label
| P(i) = |phi_i|^eta / sum...      |  <- LaTeX rendered
| eta = 2.0                        |  <- Current value(s)
|                                  |
| Branch Probability               |
| P_branch = ...                   |
|                                  |
+----------------------------------+
| RETURN STROKE                    |
|                                  |
| Current Waveform (Heidler)       |
| I(t) = ...                       |
|                                  |
| Luminosity Decay                 |
| L(t) = L_0 * e^(-t/tau)          |
+----------------------------------+
```

---

## 3. Interactive Parameter Controls

### 3.1 Parameters to Expose

| Parameter | Range | Default | Formula Link | Visual Impact |
|-----------|-------|---------|--------------|---------------|
| eta (branching exponent) | 1.0 - 4.0 | 2.0 | Growth Probability | Most dramatic: controls branch density |
| stepLength | 0.01 - 0.08 | 0.03 | - | Segment coarseness |
| branchAngle | 10 - 45 | 20 | Branch Probability | Spread of branches |
| maxBranchDepth | 1 - 5 | 3 | - | Complexity of tree |
| returnStrokeSpeed | 0.1 - 1.0 | 0.3 | Return Stroke | Visible upward wave |
| strokeCount | 1 - 8 | 4 | - | Number of flashes |
| detail (existing) | 0.2 - 2.0 | 1.0 | - | Resolution |
| speed (existing) | 0.1 - 2.0 | 1.0 | - | Playback speed |

### 3.2 Parameter Slider Component

```tsx
// components/ParameterSlider.tsx
interface ParameterSliderProps {
  id: string;
  label: string;
  value: number;
  min: number;
  max: number;
  step: number;
  unit?: string;
  linkedFormula?: string;
  linkedVariable?: string;
  onChange: (value: number) => void;
  onActiveChange?: (isActive: boolean) => void;
}

export const ParameterSlider: React.FC<ParameterSliderProps> = ({
  id,
  label,
  value,
  min,
  max,
  step,
  unit = '',
  onChange,
  onActiveChange,
}) => {
  const handlePointerDown = () => onActiveChange?.(true);
  const handlePointerUp = () => onActiveChange?.(false);

  return (
    <div className="parameter-slider">
      <label htmlFor={id}>{label}</label>
      <input
        id={id}
        type="range"
        min={min}
        max={max}
        step={step}
        value={value}
        onChange={(e) => onChange(parseFloat(e.target.value))}
        onPointerDown={handlePointerDown}
        onPointerUp={handlePointerUp}
        onPointerLeave={handlePointerUp}
      />
      <span className="value">{value.toFixed(2)}{unit}</span>
    </div>
  );
};
```

### 3.3 Slider Placement

Place sliders within the math panel, below their corresponding formulas. Group logically:

```
+------------------------------------+
| PHYSICS PARAMETERS                 |
|                                    |
| [Growth Probability formula]       |
| eta: [=====o=======] 2.0           |
|                                    |
| [Branch Probability formula]       |
| Branch prob: [===o========] 0.03   |
| Branch decay: [======o====] 2.0    |
|                                    |
+------------------------------------+
| VISUALIZATION                      |
|                                    |
| [Return Stroke formula]            |
| Stroke speed: [====o======] 0.3c   |
| Stroke count: [=====o=====] 4      |
|                                    |
| [Luminosity formula]               |
| Decay rate: [=======o===] 100ms    |
+------------------------------------+
| RENDERING                          |
|                                    |
| Detail: [=======o===] 1.0          |
| Speed:  [====o======] 1.0x         |
+------------------------------------+
```

Alternative: Keep sliders in a bottom bar if panel becomes too cluttered. The math panel shows formulas only; sliders remain at bottom but grouped more logically:

```
[eta] [branchProb] [depth] | [strokeSpeed] [count] | [detail] [speed]
 Physics                     Animation               Rendering
```

---

## 4. Parameter-to-Formula Connection

### 4.1 State Management

Use React context to share parameter state between sliders and formula displays:

```tsx
// context/LightningContext.tsx
interface LightningParams {
  eta: number;
  stepLength: number;
  branchAngle: number;
  maxBranchDepth: number;
  returnStrokeSpeed: number;
  strokeCount: number;
  detail: number;
  speed: number;
}

interface LightningContextValue {
  params: LightningParams;
  setParam: (key: keyof LightningParams, value: number) => void;
  activeParam: keyof LightningParams | null;
  setActiveParam: (key: keyof LightningParams | null) => void;
  triggerNewBolt: () => void;
}

const LightningContext = createContext<LightningContextValue | null>(null);

export const LightningProvider: React.FC<PropsWithChildren> = ({ children }) => {
  const [params, setParams] = useState<LightningParams>(DEFAULT_PARAMS);
  const [activeParam, setActiveParam] = useState<keyof LightningParams | null>(null);

  const setParam = useCallback((key: keyof LightningParams, value: number) => {
    setParams(prev => ({ ...prev, [key]: value }));
  }, []);

  const triggerNewBolt = useCallback(() => {
    // Emit event or callback to Scene to regenerate bolt
  }, []);

  return (
    <LightningContext.Provider value={{ params, setParam, activeParam, setActiveParam, triggerNewBolt }}>
      {children}
    </LightningContext.Provider>
  );
};
```

### 4.2 Highlight Behavior

When a slider is being dragged:

1. **Slider state**: `onPointerDown` sets `activeParam` to the slider's parameter key
2. **Formula lookup**: Each formula knows which variables link to which parameters
3. **Highlight application**: FormulaDisplay checks if any of its variables link to `activeParam`
4. **Visual feedback**: The formula container gets the `.highlighted` class; the specific variable in the LaTeX turns cyan

```tsx
// Inside MathPanel component
const { activeParam } = useLightningContext();

const getHighlightedVars = (formula: Formula): string[] => {
  return formula.variables
    .filter(v => v.linkedParameter === activeParam)
    .map(v => v.name);
};

return (
  <div className="math-panel">
    {FORMULAS.map(formula => (
      <FormulaDisplay
        key={formula.id}
        latex={formula.latex}
        label={formula.label}
        highlightVars={getHighlightedVars(formula)}
        className={
          formula.variables.some(v => v.linkedParameter === activeParam)
            ? 'highlighted'
            : ''
        }
        currentValues={getValuesForFormula(formula, params)}
      />
    ))}
  </div>
);
```

### 4.3 Value Display

Show the current numeric value next to or below the symbolic variable:

**Option A**: Inline substitution (e.g., `eta = 2.0` shown below the formula)
**Option B**: Tooltip on hover over the variable
**Option C**: Side annotation (e.g., `P(i) = |phi|^eta` with `eta = 2.0` to the right)

Recommendation: **Option A** is clearest. Display as small chips below the formula.

### 4.4 Trigger New Bolt on Parameter Change

When a slider value changes:
- Debounce the callback (300ms) to avoid regenerating on every drag tick
- After debounce, call `triggerNewBolt()` which signals the Scene to create a new bolt with updated params
- The bolt uses the new parameters from context

```tsx
const debouncedTrigger = useMemo(
  () => debounce(() => triggerNewBolt(), 300),
  [triggerNewBolt]
);

const handleParamChange = (key: keyof LightningParams, value: number) => {
  setParam(key, value);
  debouncedTrigger();
};
```

---

## 5. Animation Coordination

### 5.1 Panel Entrance Animation

On page load, formulas fade in sequentially (staggered):

```css
.formula-display {
  opacity: 0;
  transform: translateX(-20px);
  animation: slideIn 0.5s ease forwards;
}

.formula-display:nth-child(1) { animation-delay: 0.1s; }
.formula-display:nth-child(2) { animation-delay: 0.2s; }
.formula-display:nth-child(3) { animation-delay: 0.3s; }
.formula-display:nth-child(4) { animation-delay: 0.4s; }
.formula-display:nth-child(5) { animation-delay: 0.5s; }

@keyframes slideIn {
  to {
    opacity: 1;
    transform: translateX(0);
  }
}
```

### 5.2 Bolt-Formula Sync

When a new bolt fires (return stroke begins), briefly pulse the "Growth Probability" formula:

```tsx
// In MathPanel
const [pulsingFormula, setPulsingFormula] = useState<string | null>(null);

useEffect(() => {
  const handleBoltFire = () => {
    setPulsingFormula('growth-probability');
    setTimeout(() => setPulsingFormula(null), 500);
  };

  window.addEventListener('bolt:fire', handleBoltFire);
  return () => window.removeEventListener('bolt:fire', handleBoltFire);
}, []);

// Apply pulse class
<FormulaDisplay
  className={pulsingFormula === formula.id ? 'pulsing' : ''}
  ...
/>
```

```css
.formula-display.pulsing {
  animation: pulse 0.5s ease;
}

@keyframes pulse {
  0%, 100% { box-shadow: none; }
  50% { box-shadow: 0 0 20px rgba(79, 209, 197, 0.4); }
}
```

### 5.3 Keep Panel Static During Animation

The math panel should not animate or shift while the bolt is playing. All panel animations (entrance, highlights, pulses) should be subtle and brief. The user's focus should remain on the lightning.

---

## 6. Component Structure

### 6.1 Component Tree

```
ShowcasePage
├── LightningProvider (context)
│   ├── Canvas (r3f)
│   │   └── Scene
│   │       ├── GroundPlane
│   │       ├── LightningController
│   │       └── CloudGrid
│   ├── MathPanel
│   │   ├── FormulaSection (Stepped Leader Physics)
│   │   │   ├── FormulaDisplay (Growth Probability)
│   │   │   │   └── ParameterSlider (eta)
│   │   │   └── FormulaDisplay (Branch Probability)
│   │   │       └── ParameterSlider (branchProb)
│   │   ├── FormulaSection (Return Stroke)
│   │   │   ├── FormulaDisplay (Heidler Current)
│   │   │   └── FormulaDisplay (Luminosity Decay)
│   │   └── ParameterGroup (Rendering)
│   │       ├── ParameterSlider (detail)
│   │       └── ParameterSlider (speed)
│   ├── ParameterControls (bottom bar, alternative placement)
│   └── NavigationIcons
```

### 6.2 File Structure

```
client/src/
├── components/
│   ├── math/
│   │   ├── FormulaDisplay.tsx
│   │   ├── FormulaSection.tsx
│   │   └── index.ts
│   ├── controls/
│   │   ├── ParameterSlider.tsx
│   │   ├── ParameterControls.tsx
│   │   └── index.ts
│   └── ...
├── context/
│   └── LightningContext.tsx
├── config/
│   └── formulas.ts
├── pages/
│   └── ShowcasePage/
│       ├── ShowcasePage.tsx
│       ├── ShowcasePage.css
│       ├── MathPanel.tsx
│       ├── MathPanel.css
│       └── Scene.tsx
└── ...
```

### 6.3 Communication Pattern

1. **Props down**: ShowcasePage passes params to Scene via props (or Scene reads from context)
2. **Context for shared state**: LightningContext holds params, activeParam, and triggerNewBolt
3. **Events for animations**: Use custom events (`bolt:fire`) for cross-component animation triggers
4. **No refs needed**: All state flows through context/props

---

## 7. Existing Slider Migration

### 7.1 Current Sliders

The existing `detail` and `speed` sliders in ShowcasePage.tsx should be:
- Removed from the inline `.controls` div
- Added to the LightningContext state
- Rendered within the new MathPanel or ParameterControls component

### 7.2 Mapping to Physics

| Existing Slider | Physics Meaning | Placement |
|-----------------|-----------------|-----------|
| Detail (0.2-2.0) | Simulation resolution / segment count | Rendering section |
| Speed (0.1-2.0x) | Animation playback multiplier | Rendering section |

These don't correspond to specific formulas, so they go in a "Rendering" group at the bottom of the panel.

### 7.3 Migration Steps

1. Add `detail` and `speed` to `LightningParams` in context
2. Remove the `.controls` div from ShowcasePage
3. Add the sliders to MathPanel's "Rendering" section
4. Update Scene to read from context instead of props

---

## 8. Implementation Order

### Phase 1: Layout Foundation
1. Create LightningContext with basic params
2. Restructure ShowcasePage layout (Canvas positioning, MathPanel div)
3. Implement gradient fade CSS for MathPanel
4. Shift animation right (camera or group offset)

### Phase 2: Math Rendering
1. Install KaTeX, create FormulaDisplay component
2. Define formula data in config/formulas.ts
3. Create MathPanel component with static formula display
4. Style for dark theme

### Phase 3: Interactive Controls
1. Create ParameterSlider component
2. Add slider rendering within MathPanel
3. Wire sliders to context state
4. Migrate existing Detail/Speed sliders

### Phase 4: Highlight System
1. Add activeParam tracking to context
2. Implement highlight logic in FormulaDisplay
3. Add CSS transitions for highlights
4. Connect slider pointerDown/Up to activeParam

### Phase 5: Animation Polish
1. Add entrance animations for formulas
2. Implement bolt-fire pulse effect
3. Add debounced bolt regeneration on param change
4. Responsive breakpoints

---

## 9. Open Questions / Future Refinements

1. **Slider placement**: Within panel next to formulas, or consolidated bottom bar?
2. **Formula detail level**: Show all variables, or only the key user-adjustable ones?
3. **Mobile experience**: Math panel as collapsible drawer? Swipe to reveal?
4. **Preset configurations**: Buttons for "Realistic", "Dense Branching", "Minimal"?
5. **Animation timing**: How long should the pulse last? Should it sync with return stroke duration?

---

*Plan authored for lightning visualization showcase page.*
*Implementation target: modular, maintainable, visually cohesive.*
