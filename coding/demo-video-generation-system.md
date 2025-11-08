# Automated Demo Video Generation System - Implementation Guide

## Purpose
This prompt template provides a comprehensive guide for creating an automated demo video generation system that records HD tutorial videos of user workflows with visible cursor tracking, proper pacing, and document export functionality.

Use this as a reference when implementing demo video generation for web applications.

## Usage
Copy relevant sections when setting up automated demo video generation. Adapt the code examples to your specific application structure.

---

## The Prompt

# Automated Demo Video Generation System - Implementation Guide

## Context
This guide documents best practices for creating an automated demo video generation system that records HD tutorial videos of user workflows with visible cursor tracking, proper pacing, and document export functionality.

Use this as a template when implementing demo video generation for similar applications.

## System Requirements

### 1. Speed Configuration System
Implement a dynamic speed multiplier system with these modes:
- **FAST** (0.2x): Minimal delays, for testing and quick generation (~2-3 min/test)
- **NORMAL** (1.0x): Standard timing, balanced speed (~5-7 min/test)
- **SLOW** (2.0x): Human-paced tutorials, for final videos (~10-15 min/test)

**Critical Implementation:**
```typescript
// demos/helpers/demo-config.ts
const SPEED_MULTIPLIERS = { fast: 0.2, normal: 1.0, slow: 2.0 }
const DEMO_SPEED = process.env.DEMO_SPEED || 'normal'
const SPEED_MULTIPLIER = SPEED_MULTIPLIERS[DEMO_SPEED]

export const wait = (ms: number) => ms * SPEED_MULTIPLIER
export const typeDelay = (ms: number) => ms * SPEED_MULTIPLIER
export const timeout = (baseMs: number) => (baseMs * SPEED_MULTIPLIER) + 180000 // +3min buffer

export function logSpeedConfig() {
  console.log(`⚡ Demo Speed Mode: ${DEMO_SPEED} (${SPEED_MULTIPLIER}x)`)
  console.log(`   Set DEMO_SPEED=fast|normal|slow to change speed`)
}
```

### 2. Test Structure Template

Each export test MUST follow this structure:

```typescript
import { test, expect } from '@playwright/test'
import { enableCursorTracking } from '../helpers/cursor-tracker'
import { wait, typeDelay, timeout, logSpeedConfig } from '../helpers/demo-config'

test('[Feature] with Export Demo', async ({ page }) => {
  // 1. Dynamic timeout based on speed
  test.setTimeout(timeout(300000)) // Base: 5 minutes

  // 2. Log configuration
  console.log('\n🎬 ========================================')
  console.log('   [FEATURE NAME] & EXPORT')
  console.log('========================================\n')
  logSpeedConfig()

  // 3. Enable cursor tracking
  console.log('🔴 Enabling enhanced cursor tracker...')
  await enableCursorTracking(page)

  // 4. Load prerequisite data from previous steps
  console.log('📥 Loading prerequisite data...')
  await page.goto('/')
  await page.evaluate(() => {
    localStorage.setItem('scopeData', JSON.stringify({...}))
    localStorage.setItem('risks', JSON.stringify([...]))
    // Load ALL data from previous steps
  })

  // 5. Navigate and perform workflow
  await page.goto('/feature')
  await page.waitForTimeout(wait(1000))

  // 6. Wrap ALL waits with wait()
  await page.waitForTimeout(wait(500))
  await input.pressSequentially('text', { delay: typeDelay(50) })

  // 7. Export workflow
  const downloadPromise = page.waitForEvent('download')
  await exportButton.click()
  const download = await downloadPromise
  await download.saveAs(`test-results/${filename}`)

  // 8. PDF conversion
  const pdfFilename = filename.replace('.docx', '.pdf')
  execSync(`/opt/homebrew/bin/soffice --headless --convert-to pdf --outdir "${outputDir}" "${docxPath}"`)

  // 9. PDF viewing with error handling (CRITICAL!)
  let httpServer: any = null
  try {
    httpServer = spawn('python3', ['-m', 'http.server', '8899', '--directory', outputDir])
    await page.waitForTimeout(3000) // Server startup (don't scale)

    await page.goto(`http://localhost:8899/${pdfFilename}`, { timeout: 10000 })
    await page.waitForLoadState('networkidle', { timeout: 10000 })

    // Scroll through PDF
    for (let i = 0; i < 35; i++) {
      await page.keyboard.press('Space')
      await page.waitForTimeout(wait(250))
    }
  } catch (error: any) {
    console.log('⚠️  PDF viewing skipped (headless mode limitation)')
    console.log('   Note: PDF was created successfully but cannot be displayed in headless browser\n')
  } finally {
    if (httpServer) {
      httpServer.kill()
      console.log('✅ HTTP server stopped\n')
    }
  }
})
```

### 3. Common Component Bugs to Fix

**Bug 1: Organization Name Default**
```typescript
// ❌ WRONG - Causes PDF conversion failures
const organizationName = scopeData?.organizationName || '[Organization Name]'

// ✅ CORRECT
const organizationName = scopeData?.organizationName || 'organization'
```

**Bug 2: Array Spreading for docx Library**
```typescript
// ❌ WRONG - Creates invalid XML
sections.push(createGenerationFooter())

// ✅ CORRECT
sections.push(...createGenerationFooter())
```

**Bug 3: localStorage Data Structure**
```typescript
// ❌ WRONG - Wrapped in object
localStorage.setItem('data', JSON.stringify({ items: [...] }))

// ✅ CORRECT - Direct array
localStorage.setItem('data', JSON.stringify([...]))
```

### 4. Video Generation Strategy

**NEVER run slow mode first!** Use this workflow:

#### Phase 1: Fast Mode Testing (Iterative)
```bash
#!/bin/bash
export DEMO_SPEED=fast

# Test each step individually
npx playwright test demos/step-01/test.spec.ts --headed --project=chromium --workers=1

# Fix issues immediately
# Repeat until ALL tests pass
```

#### Phase 2: Slow Mode Generation (Final)
```bash
#!/bin/bash
export DEMO_SPEED=slow
VIDEO_DIR="tutorial-videos"

# Only run once ALL tests pass in fast mode
# Run headed so you can watch
# Copy videos to organized folder structure
```

### 5. Generation Script Template

```bash
#!/bin/bash
export DEMO_SPEED=${DEMO_SPEED:-slow}
VIDEO_DIR="tutorial-videos"

echo "Generating videos in $DEMO_SPEED mode..."
rm -rf test-results "$VIDEO_DIR"
mkdir -p test-results "$VIDEO_DIR"

copy_video() {
  local step_name=$1
  local video_file=$(find test-results -name "video.webm" -type f | head -1)

  if [ -f "$video_file" ]; then
    cp "$video_file" "$VIDEO_DIR/${step_name}.webm"
    echo "   ✅ Copied: ${step_name}.webm ($(du -h "$VIDEO_DIR/${step_name}.webm" | cut -f1))"
  fi
  rm -rf test-results/*
}

# Generate each step
npx playwright test demos/step-01/test.spec.ts --headed --project=chromium --workers=1
copy_video "step-01-feature-name"

# ... repeat for all steps
```

### 6. Playwright Configuration

```typescript
// playwright.config.ts
export default defineConfig({
  use: {
    video: 'on',
    viewport: { width: 1920, height: 1080 },
    launchOptions: {
      slowMo: 0, // Don't use slowMo, use wait() instead
    }
  },
})
```

### 7. Data Flow Architecture

```
Step 1 (Scope)
  ↓ scopeData
Step 2 (Policy)
  ↓ policyData
Step 3 (Objectives)
  ↓ objectives
Step 4 (Risks)
  ↓ risks
Step 5 (Treatments)
  ↓ riskTreatments
Step 6 (SOA)
  ↓ statementOfApplicability
Step 7 (Plans)
  ↓ riskTreatmentPlans
```

Each step MUST load ALL previous data in localStorage.

### 8. Critical Rules

1. **Always wrap timeouts**: `wait(500)` not `500`
2. **Always wrap typing delays**: `typeDelay(50)` not `50`
3. **Always scale test timeouts**: `timeout(300000)` not `300000`
4. **Never scale server startup**: `await page.waitForTimeout(3000)` for HTTP server
5. **Always handle PDF viewing errors**: Use try-catch-finally
6. **Test fast first**: Never run slow mode until fast passes
7. **Load prerequisite data**: Every test needs prior steps' data
8. **Use headed mode for final generation**: So you can watch

### 9. Skip Complex Workflow Tests

Some tests are too complex for automated video generation:
```bash
# Skip tests that require extensive user interaction
if [ -f "demos/complex-workflow/test.spec.ts" ]; then
  echo "⏭️  Skipping complex workflow - requires manual demonstration"
fi
```

### 10. Success Criteria

Before running slow mode, verify:
- ✅ All tests pass in fast mode
- ✅ All Word exports work
- ✅ All PDF conversions succeed
- ✅ No data loading errors
- ✅ No timeout failures
- ✅ Videos are created in test-results

## Implementation Checklist

- [ ] Create `demos/helpers/demo-config.ts` with speed system
- [ ] Create cursor tracker helper
- [ ] Write export tests following template
- [ ] Fix component bugs (org name, array spreading)
- [ ] Test all steps in fast mode
- [ ] Fix any failures
- [ ] Re-test in fast mode until all pass
- [ ] Run final slow mode generation
- [ ] Verify all videos in output folder

## Expected Results

- **Fast mode**: 11 steps in ~30-40 minutes
- **Slow mode**: 11 steps in ~1-2 hours
- **Output**: HD 1920x1080 WebM videos, human-paced tutorials
- **Success rate**: 100% (after fast mode validation)

## Lessons Learned from ISMS Implementation

### 1. PDF Viewing in Headless Mode
Headless browsers trigger downloads for PDFs instead of displaying them. Always wrap PDF viewing in try-catch:
```typescript
try {
  await page.goto(`http://localhost:8899/${pdfFilename}`, { timeout: 10000 })
  // ... PDF viewing code
} catch (error) {
  console.log('⚠️  PDF viewing skipped (headless mode limitation)')
}
```

### 2. Slow Mode Takes Time
- SLOW mode at 2x multiplier can take 1-2 hours for complete generation
- This is EXPECTED and correct behavior for human-paced tutorials
- Always test in FAST mode first to validate everything works

### 3. Step 5 Risk Treatment Pattern
Tests that require complex UI interaction with dynamically loaded data are challenging. Consider:
- Creating simplified export-only tests
- Pre-loading data instead of UI interaction
- Skipping if too complex

### 4. Video Organization
Use a helper function to copy videos from `test-results/` to organized output folder:
```bash
copy_video() {
  local step_name=$1
  local video_file=$(find test-results -name "video.webm" -type f | head -1)
  if [ -f "$video_file" ]; then
    cp "$video_file" "$VIDEO_DIR/${step_name}.webm"
  fi
  rm -rf test-results/*
}
```

### 5. Test Order Matters
Steps must run in sequence if they depend on data from previous steps. Cannot parallelize when data flow exists.

---

**Use this guide when setting up demo video generation for any similar application.**

## Quick Start for New Projects

1. Copy `demos/helpers/demo-config.ts` to your project
2. Copy cursor tracker implementation
3. Create first test using template above
4. Run in fast mode: `DEMO_SPEED=fast npx playwright test ...`
5. Fix until it passes
6. Repeat for all features
7. Final generation: `DEMO_SPEED=slow ./generate-videos.sh`

## Support

For issues or questions about this system, refer to the ISMS implementation in this repository as a complete working example.
