# KanaDojo Hybrid Modular Architecture Migration Plan

**Version:** 1.0
**Date:** 2025-12-31
**Target Architecture:** Hybrid Modular (Feature-Based + Selective FSD Principles)
**Estimated Timeline:** 5-7 days (40-56 hours)
**Risk Level:** Medium

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Migration Goals](#migration-goals)
3. [Architecture Overview](#architecture-overview)
4. [Phase 0: Preparation](#phase-0-preparation-day-0)
5. [Phase 1: Create Facades](#phase-1-create-facades-days-1-2)
6. [Phase 2: Create Widgets](#phase-2-create-widgets-days-2-3)
7. [Phase 3: Migrate Shared Components](#phase-3-migrate-shared-components-day-4)
8. [Phase 4: Add Barrel Exports & Enforcement](#phase-4-add-barrel-exports--enforcement-day-5)
9. [Phase 5: Testing & Validation](#phase-5-testing--validation-days-5-6)
10. [Rollback Strategy](#rollback-strategy)
11. [Success Metrics](#success-metrics)
12. [Post-Migration Checklist](#post-migration-checklist)

---

## Executive Summary

### Current State
- **Total Layer Violations:** 27 files in shared/ importing from features/
- **Code Duplication:** 540+ lines across Kana/Kanji/Vocabulary game components
- **Hub Pattern Anti-Pattern:** Progress/useStatsStore imported by 25+ files
- **Missing Abstractions:** No shared game orchestration, no unified facades
- **Missing Public APIs:** 8 features lack barrel exports (index.ts)

### Target State
- **Layer Violations:** 0 (enforced by ESLint)
- **Code Duplication:** < 100 lines (90% reduction via TrainingGame widget)
- **Decoupled Stats:** Event-based system, < 10 direct imports
- **Unified Widgets:** 5 major widgets abstracting cross-feature composition
- **Complete Public APIs:** All features expose clean barrel exports

### Migration Strategy
**Hybrid Modular Architecture:**
- **Features remain** as primary organizational unit
- **Widgets layer** added for complex UI compositions
- **Facades** abstract feature store access for cross-feature needs
- **Strict enforcement** via ESLint import rules
- **No entities layer** (too heavyweight for current scope)

---

## Migration Goals

### Primary Goals
1. **Eliminate Layer Violations:** shared/ must never import from features/
2. **Eliminate Duplication:** Abstract game logic into shared widgets
3. **Decouple Stats System:** Break Progress store hub pattern
4. **Create Clean APIs:** All features expose public barrel exports
5. **Enforce Boundaries:** ESLint rules prevent future violations

### Secondary Goals
6. **Improve DX:** Faster navigation with clear boundaries
7. **Enable Testing:** Isolated facades easier to mock
8. **Prepare for Scale:** Architecture supports 50+ features
9. **Maintain Performance:** No runtime overhead from abstraction
10. **OSS-Friendly:** Simpler mental model for contributors

---

## Architecture Overview

### New Directory Structure
```
kanadojo/
├── app/                    # Next.js App Router (unchanged)
│
├── widgets/                # ✨ NEW: Complex UI compositions
│   ├── TrainingGame/       # Unified game orchestrator
│   ├── MenuSystem/         # DojoMenu + GameModes + ActionBar
│   ├── GameUI/             # Stats, Progress, Exit screens
│   ├── SelectionUI/        # Content selection widgets
│   └── index.ts            # Barrel export
│
├── features/               # Feature modules (enhanced with facades)
│   ├── Kana/
│   │   ├── adapters/       # ✨ NEW: ContentAdapter implementation
│   │   ├── facade/         # ✨ NEW: Public hooks API
│   │   ├── components/     # Feature-specific UI
│   │   ├── store/          # Internal state (not exported)
│   │   ├── data/           # Internal data (not exported)
│   │   ├── lib/            # Internal utilities (not exported)
│   │   └── index.ts        # ✨ NEW: Public API only
│   ├── Kanji/
│   │   └── [same structure]
│   ├── Vocabulary/
│   │   └── [same structure]
│   ├── Progress/
│   │   ├── facade/         # ✨ NEW: Stats facade
│   │   ├── components/
│   │   ├── store/          # Private
│   │   └── index.ts        # ✨ NEW
│   └── Preferences/
│       ├── facade/         # ✨ NEW: Preferences facade
│       └── index.ts        # ✨ NEW
│
├── shared/                 # ✨ STRICT: No feature imports allowed
│   ├── components/         # Simple, reusable UI (Button, Card, etc.)
│   ├── hooks/              # Generic hooks only (no feature coupling)
│   ├── lib/                # Pure utilities
│   ├── events/             # ✨ NEW: Event bus for cross-cutting concerns
│   └── types/              # Shared type definitions
│
└── core/                   # Infrastructure (unchanged)
```

### Layer Rules

| Layer | Can Import From | Cannot Import From | Enforced By |
|-------|----------------|-------------------|-------------|
| `app/` | widgets, features (via index.ts), shared, core | feature internals | ESLint |
| `widgets/` | features (via facade), shared, core | feature stores directly | ESLint |
| `features/` | other features (via index.ts), shared, core | other feature internals | ESLint |
| `shared/` | shared, core | features, widgets | ESLint ✅ |
| `core/` | core only | app, widgets, features, shared | ESLint |

---

## Phase 0: Preparation (Day 0)

### 0.1 Create Feature Branch
```bash
git checkout -b refactor/hybrid-modular-architecture
```

### 0.2 Document Current Behavior
**Test coverage checkpoint:**
```bash
npm run test
# Record current test results
```

**Manual test scenarios:**
- [ ] Play Kana game (Pick mode)
- [ ] Play Kanji game (Input mode)
- [ ] Play Vocabulary game (Reverse-Pick mode)
- [ ] Unlock an achievement
- [ ] View stats in progress page
- [ ] Change theme in preferences
- [ ] Play Blitz mode for all 3 types
- [ ] Start Gauntlet mode

**Screenshot critical pages:**
- Home page
- Kana selection
- Game in progress
- Game exit screen
- Progress page
- Achievements page

### 0.3 Create Backup Point
```bash
git add -A
git commit -m "chore: pre-migration checkpoint

Create backup before Hybrid Modular migration

- All tests passing
- Manual testing complete
- Screenshots captured
- Ready for Phase 1"
```

### 0.4 Set Up Migration Tracking
```bash
# Create tracking file
touch MIGRATION_PROGRESS.md
```

---

## Phase 1: Create Facades (Days 1-2)

**Goal:** Abstract feature store access behind clean hook APIs

### 1.1 Create Event System (2 hours)

**File:** `shared/events/statsEvents.ts`
```typescript
// ============================================================================
// Stats Event System - Decouples game features from Progress store
// ============================================================================

export type StatEventType = 'correct' | 'incorrect' | 'session_complete';

export interface StatEvent {
  type: StatEventType;
  contentType: 'kana' | 'kanji' | 'vocabulary';
  character: string;
  correctAnswer?: string;
  userAnswer?: string;
  timestamp: number;
  metadata?: {
    gameMode?: string;
    difficulty?: string;
    timeTaken?: number;
  };
}

class StatsEventBus {
  private listeners: Map<StatEventType, Set<(event: StatEvent) => void>> = new Map();

  subscribe(eventType: StatEventType, listener: (event: StatEvent) => void): () => void {
    if (!this.listeners.has(eventType)) {
      this.listeners.set(eventType, new Set());
    }
    this.listeners.get(eventType)!.add(listener);

    // Return unsubscribe function
    return () => {
      this.listeners.get(eventType)?.delete(listener);
    };
  }

  emit(event: StatEvent): void {
    const listeners = this.listeners.get(event.type);
    if (listeners) {
      listeners.forEach(listener => listener(event));
    }
  }
}

export const statsEvents = new StatsEventBus();

// Public API for game components
export const statsApi = {
  recordCorrect(
    contentType: StatEvent['contentType'],
    character: string,
    metadata?: StatEvent['metadata']
  ) {
    statsEvents.emit({
      type: 'correct',
      contentType,
      character,
      timestamp: Date.now(),
      metadata
    });
  },

  recordIncorrect(
    contentType: StatEvent['contentType'],
    character: string,
    userAnswer: string,
    correctAnswer: string,
    metadata?: StatEvent['metadata']
  ) {
    statsEvents.emit({
      type: 'incorrect',
      contentType,
      character,
      userAnswer,
      correctAnswer,
      timestamp: Date.now(),
      metadata
    });
  },

  recordSessionComplete(
    contentType: StatEvent['contentType'],
    metadata?: StatEvent['metadata']
  ) {
    statsEvents.emit({
      type: 'session_complete',
      contentType,
      character: '',
      timestamp: Date.now(),
      metadata
    });
  }
};
```

**File:** `shared/events/achievementEvents.ts`
```typescript
// ============================================================================
// Achievement Event System
// ============================================================================

export interface AchievementEvent {
  type: 'check' | 'unlock';
  achievementId?: string;
  timestamp: number;
}

class AchievementEventBus {
  private listeners: Set<(event: AchievementEvent) => void> = new Set();

  subscribe(listener: (event: AchievementEvent) => void): () => void {
    this.listeners.add(listener);
    return () => {
      this.listeners.delete(listener);
    };
  }

  emit(event: AchievementEvent): void {
    this.listeners.forEach(listener => listener(event));
  }
}

export const achievementEvents = new AchievementEventBus();

export const achievementApi = {
  triggerCheck() {
    achievementEvents.emit({ type: 'check', timestamp: Date.now() });
  },

  recordUnlock(achievementId: string) {
    achievementEvents.emit({ type: 'unlock', achievementId, timestamp: Date.now() });
  }
};
```

**File:** `shared/events/index.ts`
```typescript
export { statsEvents, statsApi } from './statsEvents';
export type { StatEvent, StatEventType } from './statsEvents';

export { achievementEvents, achievementApi } from './achievementEvents';
export type { AchievementEvent } from './achievementEvents';
```

**Checkpoint:**
```bash
npm run check
git add shared/events/
git commit -m "feat(events): add stats and achievement event system

- Create StatsEventBus for decoupled stat tracking
- Create AchievementEventBus for achievement checks
- Public APIs: statsApi, achievementApi
- Preparation for facade layer"
```

---

### 1.2 Create Progress Facade (3 hours)

**File:** `features/Progress/facade/useGameStats.ts`
```typescript
'use client';

import { useEffect } from 'react';
import { statsApi, statsEvents } from '@/shared/events';
import type { StatEvent } from '@/shared/events';
import useStatsStore from '../store/useStatsStore';

/**
 * Game Stats Facade - Public API for stat tracking
 *
 * This facade decouples game components from the internal Progress store.
 * Games emit events; this facade subscribes and updates the store.
 */

export interface GameStats {
  correctAnswers: number;
  wrongAnswers: number;
  currentStreak: number;
  bestStreak: number;
  totalSessions: number;
}

export interface GameStatsActions {
  recordCorrect: typeof statsApi.recordCorrect;
  recordIncorrect: typeof statsApi.recordIncorrect;
  recordSessionComplete: typeof statsApi.recordSessionComplete;
  getStats: () => GameStats;
  resetSessionStats: () => void;
}

/**
 * Hook for game components to track stats
 *
 * @example
 * const stats = useGameStats();
 * stats.recordCorrect('kana', 'あ');
 */
export function useGameStats(): GameStatsActions {
  const store = useStatsStore();

  // Subscribe to stat events and update store
  useEffect(() => {
    const unsubCorrect = statsEvents.subscribe('correct', (event: StatEvent) => {
      store.incrementCorrectAnswers();
      store.updateCharacterHistory(event.character, true, event.contentType);
    });

    const unsubIncorrect = statsEvents.subscribe('incorrect', (event: StatEvent) => {
      store.incrementWrongAnswers();
      store.updateCharacterHistory(event.character, false, event.contentType);
    });

    const unsubSession = statsEvents.subscribe('session_complete', () => {
      store.saveSession();
    });

    return () => {
      unsubCorrect();
      unsubIncorrect();
      unsubSession();
    };
  }, [store]);

  return {
    recordCorrect: statsApi.recordCorrect,
    recordIncorrect: statsApi.recordIncorrect,
    recordSessionComplete: statsApi.recordSessionComplete,
    getStats: () => ({
      correctAnswers: store.correctAnswers,
      wrongAnswers: store.wrongAnswers,
      currentStreak: store.currentStreak,
      bestStreak: store.bestStreak,
      totalSessions: store.sessions.length
    }),
    resetSessionStats: store.resetSessionStats
  };
}
```

**File:** `features/Progress/facade/useStatsDisplay.ts`
```typescript
'use client';

import useStatsStore from '../store/useStatsStore';
import type { IStats } from '../store/useStatsStore';

/**
 * Read-only stats access for display components
 */
export function useStatsDisplay() {
  const stats = useStatsStore(state => ({
    correctAnswers: state.correctAnswers,
    wrongAnswers: state.wrongAnswers,
    currentStreak: state.currentStreak,
    bestStreak: state.bestStreak,
    sessions: state.sessions,
    stars: state.stars,
    characterHistory: state.characterHistory,
    statsOn: state.statsOn
  }));

  return stats;
}

/**
 * Read-only session stats
 */
export function useSessionStats() {
  return useStatsStore(state => ({
    sessionCorrect: state.correctAnswers,
    sessionWrong: state.wrongAnswers,
    sessionStreak: state.currentStreak
  }));
}
```

**File:** `features/Progress/facade/index.ts`
```typescript
export { useGameStats } from './useGameStats';
export type { GameStats, GameStatsActions } from './useGameStats';

export { useStatsDisplay, useSessionStats } from './useStatsDisplay';
```

**Checkpoint:**
```bash
npm run check
git add features/Progress/facade/
git commit -m "feat(progress): add Progress facade layer

- Create useGameStats() hook for stat tracking
- Create useStatsDisplay() for read-only access
- Event-based architecture decouples from store
- Public API for widgets and features"
```

---

### 1.3 Create Kana Facade (2 hours)

**File:** `features/Kana/facade/useKanaSelection.ts`
```typescript
'use client';

import useKanaStore from '../store/useKanaStore';
import type { IKanaState } from '../store/useKanaStore';

/**
 * Kana Selection Facade - Public API for selection state
 */

export interface KanaSelection {
  selectedGroupIndices: number[];
  totalSelected: number;
  isEmpty: boolean;
}

export interface KanaSelectionActions {
  addGroup: (index: number) => void;
  removeGroup: (index: number) => void;
  toggleGroup: (index: number) => void;
  clearSelection: () => void;
  selectAll: () => void;
  isGroupSelected: (index: number) => boolean;
}

export function useKanaSelection(): KanaSelection & KanaSelectionActions {
  const store = useKanaStore();

  return {
    // State
    selectedGroupIndices: store.kanaGroupIndices,
    totalSelected: store.kanaGroupIndices.length,
    isEmpty: store.kanaGroupIndices.length === 0,

    // Actions
    addGroup: store.addKanaGroupIndices,
    removeGroup: store.removeKanaGroupIndex,
    toggleGroup: (index: number) => {
      if (store.kanaGroupIndices.includes(index)) {
        store.removeKanaGroupIndex(index);
      } else {
        store.addKanaGroupIndices(index);
      }
    },
    clearSelection: store.clearKanaGroupIndices,
    selectAll: () => {
      // Logic for selecting all groups
      // (import kana data if needed)
    },
    isGroupSelected: (index: number) => store.kanaGroupIndices.includes(index)
  };
}
```

**File:** `features/Kana/facade/useKanaContent.ts`
```typescript
'use client';

import { useMemo } from 'react';
import { kana } from '../data/kana';
import type { IKana, KanaType, KanaGroup } from '../data/kana';
import useKanaStore from '../store/useKanaStore';

/**
 * Kana Content Facade - Access to kana data based on selection
 */

export interface KanaContent {
  selectedCharacters: IKana[];
  allGroups: KanaGroup[];
  getGroupByIndex: (index: number) => KanaGroup | undefined;
  getCharactersByType: (type: KanaType) => IKana[];
}

export function useKanaContent(): KanaContent {
  const selectedIndices = useKanaStore(state => state.kanaGroupIndices);

  const selectedCharacters = useMemo(() => {
    return selectedIndices.flatMap(index => kana[index]?.data || []);
  }, [selectedIndices]);

  return {
    selectedCharacters,
    allGroups: kana,
    getGroupByIndex: (index: number) => kana[index],
    getCharactersByType: (type: KanaType) =>
      kana.filter(g => g.type === type).flatMap(g => g.data)
  };
}
```

**File:** `features/Kana/facade/index.ts`
```typescript
export { useKanaSelection } from './useKanaSelection';
export type { KanaSelection, KanaSelectionActions } from './useKanaSelection';

export { useKanaContent } from './useKanaContent';
export type { KanaContent } from './useKanaContent';
```

**Repeat for Kanji and Vocabulary** (similar structure)

**Checkpoint:**
```bash
npm run check
git add features/Kana/facade/ features/Kanji/facade/ features/Vocabulary/facade/
git commit -m "feat(facades): add Kana, Kanji, Vocabulary facades

- Create selection facades for all 3 content types
- Create content facades for data access
- Unified API pattern across features
- Preparation for widget layer"
```

---

### 1.4 Create Preferences Facade (1 hour)

**File:** `features/Preferences/facade/useAudioPreferences.ts`
```typescript
'use client';

import usePreferencesStore from '../store/usePreferencesStore';

export function useAudioPreferences() {
  return usePreferencesStore(state => ({
    silentMode: state.silentMode,
    volume: state.volume ?? 1.0,
    setSilentMode: state.setSilentMode
  }));
}
```

**File:** `features/Preferences/facade/useThemePreferences.ts`
```typescript
'use client';

import usePreferencesStore from '../store/usePreferencesStore';

export function useThemePreferences() {
  return usePreferencesStore(state => ({
    theme: state.theme,
    font: state.font,
    setTheme: state.setTheme,
    setFont: state.setFont
  }));
}
```

**File:** `features/Preferences/facade/index.ts`
```typescript
export { useAudioPreferences } from './useAudioPreferences';
export { useThemePreferences } from './useThemePreferences';
```

**Checkpoint:**
```bash
npm run check
git add features/Preferences/facade/
git commit -m "feat(preferences): add Preferences facade layer

- Create audio preferences facade
- Create theme preferences facade
- Preparation for shared component migration"
```

---

## Phase 2: Create Widgets (Days 2-3)

**Goal:** Build reusable complex UI compositions

### 2.1 Create ContentAdapter Interface (1 hour)

**File:** `widgets/TrainingGame/adapters/ContentAdapter.ts`
```typescript
// ============================================================================
// Content Adapter Interface - Abstraction for different content types
// ============================================================================

export type GameMode = 'pick' | 'reverse-pick' | 'input' | 'reverse-input';

export interface ContentAdapter<T> {
  /**
   * Get the question to display to the user
   */
  getQuestion(item: T, mode: GameMode): string;

  /**
   * Get the correct answer for validation
   */
  getCorrectAnswer(item: T, mode: GameMode): string;

  /**
   * Generate wrong answer options for multiple choice
   */
  generateOptions(item: T, pool: T[], mode: GameMode, count: number): string[];

  /**
   * Validate user answer
   */
  validateAnswer(userAnswer: string, item: T, mode: GameMode): boolean;

  /**
   * Get display metadata (readings, translations, etc.)
   */
  getMetadata?(item: T): {
    primary: string;
    secondary?: string;
    readings?: string[];
  };
}
```

**File:** `widgets/TrainingGame/adapters/KanaAdapter.ts`
```typescript
import type { IKana } from '@/features/Kana/data/kana';
import type { ContentAdapter, GameMode } from './ContentAdapter';

export const kanaAdapter: ContentAdapter<IKana> = {
  getQuestion(kana: IKana, mode: GameMode): string {
    return mode.includes('reverse') ? kana.romanization : kana.character;
  },

  getCorrectAnswer(kana: IKana, mode: GameMode): string {
    return mode.includes('reverse') ? kana.character : kana.romanization;
  },

  generateOptions(kana: IKana, pool: IKana[], mode: GameMode, count: number): string[] {
    const correct = this.getCorrectAnswer(kana, mode);
    const wrongOptions = pool
      .filter(k => this.getCorrectAnswer(k, mode) !== correct)
      .map(k => this.getCorrectAnswer(k, mode))
      .slice(0, count - 1);

    return [correct, ...wrongOptions].sort(() => Math.random() - 0.5);
  },

  validateAnswer(userAnswer: string, kana: IKana, mode: GameMode): boolean {
    const correct = this.getCorrectAnswer(kana, mode);
    return userAnswer.toLowerCase().trim() === correct.toLowerCase().trim();
  },

  getMetadata(kana: IKana) {
    return {
      primary: kana.character,
      secondary: kana.romanization
    };
  }
};
```

**Repeat for KanjiAdapter and VocabularyAdapter**

**Checkpoint:**
```bash
npm run check
git add widgets/TrainingGame/adapters/
git commit -m "feat(widgets): add ContentAdapter abstraction

- Create ContentAdapter interface
- Implement KanaAdapter
- Implement KanjiAdapter
- Implement VocabularyAdapter
- Foundation for TrainingGame widget"
```

---

### 2.2 Create TrainingGame Widget (4 hours)

**File:** `widgets/TrainingGame/hooks/useGameEngine.ts`
```typescript
'use client';

import { useState, useCallback, useMemo } from 'react';
import { statsApi } from '@/shared/events';
import type { ContentAdapter, GameMode } from '../adapters/ContentAdapter';

export interface GameEngineConfig<T> {
  content: T[];
  mode: GameMode;
  adapter: ContentAdapter<T>;
  contentType: 'kana' | 'kanji' | 'vocabulary';
}

export interface GameState<T> {
  currentItem: T | null;
  currentIndex: number;
  options: string[];
  totalQuestions: number;
  questionsAnswered: number;
  handleAnswer: (answer: string) => Promise<boolean>;
  nextQuestion: () => void;
  isComplete: boolean;
}

export function useGameEngine<T>({
  content,
  mode,
  adapter,
  contentType
}: GameEngineConfig<T>): GameState<T> {
  const [currentIndex, setCurrentIndex] = useState(0);
  const [questionsAnswered, setQuestionsAnswered] = useState(0);

  const shuffledContent = useMemo(() => {
    return [...content].sort(() => Math.random() - 0.5);
  }, [content]);

  const currentItem = shuffledContent[currentIndex] ?? null;

  const options = useMemo(() => {
    if (!currentItem || mode.includes('input')) return [];
    return adapter.generateOptions(currentItem, shuffledContent, mode, 4);
  }, [currentItem, shuffledContent, mode, adapter]);

  const handleAnswer = useCallback(
    async (answer: string): Promise<boolean> => {
      if (!currentItem) return false;

      const isCorrect = adapter.validateAnswer(answer, currentItem, mode);
      const correctAnswer = adapter.getCorrectAnswer(currentItem, mode);
      const question = adapter.getQuestion(currentItem, mode);

      if (isCorrect) {
        statsApi.recordCorrect(contentType, question);
      } else {
        statsApi.recordIncorrect(contentType, question, answer, correctAnswer);
      }

      setQuestionsAnswered(prev => prev + 1);
      return isCorrect;
    },
    [currentItem, adapter, mode, contentType]
  );

  const nextQuestion = useCallback(() => {
    setCurrentIndex(prev => prev + 1);
  }, []);

  const isComplete = currentIndex >= shuffledContent.length;

  return {
    currentItem,
    currentIndex,
    options,
    totalQuestions: shuffledContent.length,
    questionsAnswered,
    handleAnswer,
    nextQuestion,
    isComplete
  };
}
```

**File:** `widgets/TrainingGame/TrainingGame.tsx`
```typescript
'use client';

import { useEffect } from 'react';
import { useGameEngine } from './hooks/useGameEngine';
import { statsApi } from '@/shared/events';
import type { ContentAdapter, GameMode } from './adapters/ContentAdapter';

export interface TrainingGameProps<T> {
  content: T[];
  contentType: 'kana' | 'kanji' | 'vocabulary';
  mode: GameMode;
  adapter: ContentAdapter<T>;
  onComplete?: () => void;
  children: (state: ReturnType<typeof useGameEngine<T>>) => React.ReactNode;
}

/**
 * TrainingGame Widget - Unified game orchestration
 *
 * Eliminates duplication across Kana, Kanji, Vocabulary games
 *
 * @example
 * <TrainingGame
 *   content={selectedKana}
 *   contentType="kana"
 *   mode="pick"
 *   adapter={kanaAdapter}
 * >
 *   {gameState => <GameUI {...gameState} />}
 * </TrainingGame>
 */
export function TrainingGame<T>({
  content,
  contentType,
  mode,
  adapter,
  onComplete,
  children
}: TrainingGameProps<T>) {
  const gameState = useGameEngine({ content, mode, adapter, contentType });

  useEffect(() => {
    if (gameState.isComplete) {
      statsApi.recordSessionComplete(contentType);
      onComplete?.();
    }
  }, [gameState.isComplete, contentType, onComplete]);

  return <>{children(gameState)}</>;
}
```

**File:** `widgets/TrainingGame/index.ts`
```typescript
export { TrainingGame } from './TrainingGame';
export type { TrainingGameProps } from './TrainingGame';

export { kanaAdapter } from './adapters/KanaAdapter';
export { kanjiAdapter } from './adapters/KanjiAdapter';
export { vocabularyAdapter } from './adapters/VocabularyAdapter';

export type { ContentAdapter, GameMode } from './adapters/ContentAdapter';
```

**Checkpoint:**
```bash
npm run check
git add widgets/TrainingGame/
git commit -m "feat(widgets): add TrainingGame widget

- Create useGameEngine hook for game orchestration
- Create TrainingGame render-prop component
- Eliminates 540 lines of duplication
- Unified game flow for all content types"
```

---

### 2.3 Create MenuWidget (3 hours)

**File:** `widgets/MenuSystem/MenuWidget.tsx`
```typescript
'use client';

import { useKanaSelection, useKanaContent } from '@/features/Kana/facade';
import { useKanjiSelection, useKanjiContent } from '@/features/Kanji/facade';
import { useVocabSelection, useVocabContent } from '@/features/Vocabulary/facade';
import KanaCards from '@/features/Kana/components/KanaCards';
import KanjiCards from '@/features/Kanji/components';
import VocabCards from '@/features/Vocabulary/components';

export type ContentType = 'kana' | 'kanji' | 'vocabulary';

export interface MenuWidgetProps {
  contentType: ContentType;
}

/**
 * MenuWidget - Unified menu system for content selection
 *
 * Replaces DojoMenu.tsx with facade-based approach
 */
export function MenuWidget({ contentType }: MenuWidgetProps) {
  // Use facades instead of direct store access
  const kanaSelection = useKanaSelection();
  const kanjiSelection = useKanjiSelection();
  const vocabSelection = useVocabSelection();

  switch (contentType) {
    case 'kana':
      return <KanaCards />;
    case 'kanji':
      return <KanjiCards />;
    case 'vocabulary':
      return <VocabCards />;
  }
}
```

**Checkpoint:**
```bash
npm run check
git add widgets/MenuSystem/
git commit -m "feat(widgets): add MenuWidget

- Create MenuWidget for unified content selection
- Uses facades instead of direct store imports
- Replaces DojoMenu.tsx
- Layer violations eliminated"
```

---

### 2.4 Create GameUIWidget (2 hours)

**File:** `widgets/GameUI/GameUIWidget.tsx`
```typescript
'use client';

import { useStatsDisplay } from '@/features/Progress/facade';
import { ProgressBar } from './components/ProgressBar';
import { Stars } from './components/Stars';

export interface GameUIWidgetProps {
  questionsAnswered: number;
  totalQuestions: number;
  showProgress?: boolean;
  showStars?: boolean;
}

/**
 * GameUIWidget - In-game UI elements
 *
 * Uses Progress facade instead of direct store access
 */
export function GameUIWidget({
  questionsAnswered,
  totalQuestions,
  showProgress = true,
  showStars = true
}: GameUIWidgetProps) {
  const stats = useStatsDisplay();

  return (
    <div className="game-ui">
      {showProgress && (
        <ProgressBar
          current={questionsAnswered}
          total={totalQuestions}
        />
      )}
      {showStars && <Stars count={stats.stars} />}
    </div>
  );
}
```

**Checkpoint:**
```bash
npm run check
git add widgets/GameUI/
git commit -m "feat(widgets): add GameUIWidget

- Create GameUIWidget for in-game UI
- Uses Progress facade
- Eliminates shared → Progress violations
- Clean separation of concerns"
```

---

## Phase 3: Migrate Shared Components (Day 4)

**Goal:** Update shared components to use facades instead of direct feature imports

### 3.1 Update shared/hooks/useStats.tsx (30 min)

**Before:**
```typescript
import useStatsStore from '@/features/Progress/store/useStatsStore';
import { useAchievementTrigger } from '@/features/Achievements/hooks/useAchievements';
```

**After:**
```typescript
import { useGameStats } from '@/features/Progress/facade';
import { achievementApi } from '@/shared/events';

export function useStats() {
  const stats = useGameStats();

  const incrementCorrect = (contentType, character) => {
    stats.recordCorrect(contentType, character);
    // Trigger achievement check via event
    achievementApi.triggerCheck();
  };

  const incrementWrong = (contentType, character, userAnswer, correctAnswer) => {
    stats.recordIncorrect(contentType, character, userAnswer, correctAnswer);
  };

  return { incrementCorrect, incrementWrong };
}
```

**Checkpoint:**
```bash
npm run check
git add shared/hooks/useStats.tsx
git commit -m "refactor(shared): migrate useStats to facade pattern

- Replace direct Progress store import with facade
- Use event system for achievement triggers
- Eliminates layer violation"
```

---

### 3.2 Update shared/hooks/useAudio.ts (30 min)

**Before:**
```typescript
import usePreferencesStore from '@/features/Preferences/store/usePreferencesStore';
```

**After:**
```typescript
import { useAudioPreferences } from '@/features/Preferences/facade';

export function useClick() {
  const { silentMode } = useAudioPreferences();
  // ... rest unchanged
}
```

**Checkpoint:**
```bash
npm run check
git add shared/hooks/useAudio.ts
git commit -m "refactor(shared): migrate useAudio to Preferences facade

- Replace direct store import with facade
- Cleaner separation of concerns"
```

---

### 3.3 Update shared/components/Game/* (2 hours)

**Files to update:**
- ReturnFromGame.tsx
- Stats.tsx
- ProgressBar.tsx
- Animals.tsx
- Stars.tsx

**Pattern:**
```typescript
// Before
import useStatsStore from '@/features/Progress/store/useStatsStore';
const stats = useStatsStore(state => state.correctAnswers);

// After
import { useStatsDisplay } from '@/features/Progress/facade';
const stats = useStatsDisplay();
```

**Checkpoint:**
```bash
npm run check
git add shared/components/Game/
git commit -m "refactor(shared): migrate Game components to Progress facade

- Replace all useStatsStore imports with useStatsDisplay
- 5 files updated
- Layer violations eliminated"
```

---

### 3.4 Update shared/components/Menu/* (3 hours)

**This is the most complex migration:**

**DojoMenu.tsx → Delete (replaced by MenuWidget)**

**GameModes.tsx:**
```typescript
// Before
import useKanaStore from '@/features/Kana/store/useKanaStore';
import useKanjiStore from '@/features/Kanji/store/useKanjiStore';
import useVocabStore from '@/features/Vocabulary/store/useVocabStore';

// After
import { useKanaSelection } from '@/features/Kana/facade';
import { useKanjiSelection } from '@/features/Kanji/facade';
import { useVocabSelection } from '@/features/Vocabulary/facade';
```

**TrainingActionBar.tsx:** (similar pattern)

**SelectionStatusBar.tsx:** (similar pattern)

**UnitSelector.tsx:** (similar pattern)

**Checkpoint:**
```bash
npm run check
git add shared/components/Menu/
git commit -m "refactor(shared): migrate Menu components to facades

- Replace all direct store imports with facades
- DojoMenu.tsx deleted (replaced by MenuWidget)
- 4 components updated
- All layer violations eliminated"
```

---

## Phase 4: Add Barrel Exports & Enforcement (Day 5)

**Goal:** Create public APIs and enforce boundaries

### 4.1 Add Barrel Exports to Features (3 hours)

**File:** `features/Kana/index.ts`
```typescript
// ============================================================================
// Kana Feature - Public API
// ============================================================================

// Facades (primary API)
export { useKanaSelection, useKanaContent } from './facade';
export type { KanaSelection, KanaSelectionActions, KanaContent } from './facade';

// Components (page-level)
export { default as KanaGame } from './components/Game';
export { default as KanaCards } from './components/KanaCards';
export { default as KanaBlitz } from './components/Blitz';
export { default as KanaGauntlet } from './components/Gauntlet';
export { default as SubsetDictionary } from './components/SubsetDictionary';

// Adapters
export { kanaAdapter } from './adapters/KanaAdapter';

// Types (read-only data types)
export type { IKana, KanaType, KanaGroup } from './data/kana';

// ============================================================================
// INTERNAL - DO NOT IMPORT
// ============================================================================
// - store/useKanaStore.ts (use facade instead)
// - data/kana.ts (use useKanaContent() instead)
// - lib/* (internal utilities)
```

**Repeat for:**
- `features/Kanji/index.ts`
- `features/Vocabulary/index.ts`
- `features/Progress/index.ts`
- `features/Preferences/index.ts`
- `features/Achievements/index.ts`
- `features/CrazyMode/index.ts`

**Checkpoint:**
```bash
npm run check
git add features/*/index.ts
git commit -m "feat(features): add barrel exports to all features

- Create public API via index.ts
- 8 features updated
- Clear separation of public/private
- Documentation of internal files"
```

---

### 4.2 Add ESLint Import Rules (1 hour)

**File:** `eslint.config.js`
```javascript
import { FlatCompat } from '@eslint/eslintrc';

const compat = new FlatCompat();

export default [
  ...compat.config({
    extends: ['next/core-web-vitals'],
    rules: {
      // ========================================================================
      // Layer Boundary Enforcement
      // ========================================================================
      'import/no-restricted-paths': [
        'error',
        {
          zones: [
            // Shared layer cannot import from features or widgets
            {
              target: './shared',
              from: './features',
              message:
                'shared/ cannot import from features/. Use facades or move component to widgets/.'
            },
            {
              target: './shared',
              from: './widgets',
              message: 'shared/ cannot import from widgets/. Dependency inversion violation.'
            },

            // Features cannot import from feature internals
            {
              target: './features/*/!(index).{ts,tsx}',
              from: './features/*/store',
              except: ['./features/*/store/index.ts'], // Allow facade
              message:
                'Features cannot import stores from other features. Use facade (index.ts) instead.'
            },
            {
              target: './features/*/!(index).{ts,tsx}',
              from: './features/*/data',
              except: ['./features/*/data/index.ts'], // Allow facade
              message:
                'Features cannot import data from other features directly. Use facade instead.'
            },

            // Widgets must use facades, not direct store access
            {
              target: './widgets',
              from: './features/*/store',
              message:
                'widgets/ cannot import feature stores directly. Use facades from features/*/index.ts instead.'
            },
            {
              target: './widgets',
              from: './features/*/data',
              message:
                'widgets/ cannot import feature data directly. Use facades from features/*/index.ts instead.'
            }
          ]
        }
      ]
    }
  })
];
```

**Checkpoint:**
```bash
npm run lint
# Should pass with 0 violations

git add eslint.config.js
git commit -m "feat(eslint): add layer boundary enforcement rules

- Prevent shared → features imports
- Prevent shared → widgets imports
- Prevent widgets → feature stores
- Prevent cross-feature store access
- Enforce facade usage"
```

---

### 4.3 Update App Pages to Use Barrel Exports (1 hour)

**File:** `app/[locale]/kana/page.tsx`
```typescript
// Before
import KanaCards from '@/features/Kana/components/KanaCards';

// After
import { KanaCards } from '@/features/Kana';
```

**Update all app/ pages:**
- app/[locale]/kana/page.tsx
- app/[locale]/kanji/page.tsx
- app/[locale]/vocabulary/page.tsx
- app/[locale]/progress/page.tsx
- app/[locale]/preferences/page.tsx
- app/[locale]/achievements/page.tsx

**Checkpoint:**
```bash
npm run check
npm run lint
git add app/
git commit -m "refactor(app): use barrel exports from features

- All app pages import from feature index.ts
- Consistent import pattern
- Enforced by ESLint"
```

---

## Phase 5: Testing & Validation (Days 5-6)

### 5.1 Automated Testing (2 hours)

**Run test suite:**
```bash
npm run test
# All tests should pass
```

**If tests fail:**
1. Update test imports to use barrel exports
2. Mock facades instead of stores
3. Update snapshots if UI unchanged

**Checkpoint:**
```bash
git add **/*.test.ts **/*.test.tsx
git commit -m "test: update tests for Hybrid Modular architecture

- Update imports to use barrel exports
- Mock facades instead of stores
- All tests passing"
```

---

### 5.2 Manual Testing (3 hours)

**Test checklist:**

**Kana Feature:**
- [ ] Select Hiragana groups
- [ ] Play Pick mode game
- [ ] Play Input mode game
- [ ] Play Reverse-Pick mode game
- [ ] Play Reverse-Input mode game
- [ ] Stats increment correctly
- [ ] Achievements unlock on milestones
- [ ] Blitz mode works
- [ ] Gauntlet mode works

**Kanji Feature:**
- [ ] Select JLPT N5 kanji
- [ ] Play all 4 game modes
- [ ] Stats tracked separately
- [ ] Achievements work
- [ ] Blitz mode works

**Vocabulary Feature:**
- [ ] Select JLPT N5 vocabulary
- [ ] Play all 4 game modes
- [ ] Stats tracked separately
- [ ] Achievements work
- [ ] Blitz mode works

**Cross-cutting Concerns:**
- [ ] Theme switching works (Preferences)
- [ ] Font switching works (Preferences)
- [ ] Silent mode works (Audio)
- [ ] Stats display correct in Progress page
- [ ] Achievement notifications appear
- [ ] Goal timers work

**Checkpoint:**
```bash
# If all manual tests pass
git add .
git commit -m "test: manual testing complete

All features tested:
- Kana, Kanji, Vocabulary games
- Stats tracking
- Achievements
- Preferences
- Audio system

No regressions found."
```

---

### 5.3 Performance Testing (1 hour)

**Metrics to check:**
- [ ] Initial page load time (< 2s)
- [ ] Game start time (< 500ms)
- [ ] Answer feedback delay (< 100ms)
- [ ] Theme switch time (< 200ms)
- [ ] Bundle size (no significant increase)

**Tools:**
```bash
npm run build
# Check build output for bundle size
```

**Lighthouse audit:**
- Performance score > 90
- Accessibility score > 95
- SEO score > 95

**Checkpoint:**
```bash
# If performance acceptable
git add .
git commit -m "test: performance testing complete

Metrics:
- Initial load: [X]s
- Game start: [X]ms
- Bundle size: [X]MB

All targets met."
```

---

## Rollback Strategy

### If Critical Issues Found

**Emergency Rollback:**
```bash
git reset --hard [pre-migration-commit-hash]
# or
git revert [migration-commit-range]
```

**Partial Rollback:**

If only one phase has issues, revert just that phase:
```bash
# Example: Revert Phase 3 only
git log --oneline
git revert [phase-3-commits]
```

**Incremental Rollback:**

1. Keep facades and events (Phase 1) - these are additive
2. Remove widgets (Phase 2) - delete widgets/ folder
3. Restore shared component imports (Phase 3) - revert changes
4. Remove barrel exports (Phase 4) - optional, no runtime impact

### Known Risks & Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| Type errors from facade changes | Build failure | Run `npm run check` after each phase |
| Missing facade methods | Runtime errors | Comprehensive facade testing before migration |
| ESLint rule too strict | Development friction | Adjust rules in eslint.config.js |
| Performance regression | Slower app | Profile before/after, optimize facades |
| Achievement system breaks | User frustration | Extensive testing of event system |

---

## Success Metrics

### Quantitative Metrics

| Metric | Before | Target | Actual |
|--------|--------|--------|--------|
| Layer violations (shared → features) | 27 | 0 | __ |
| Code duplication (game logic) | 540 lines | < 100 lines | __ |
| Direct Progress store imports | 25+ | < 10 | __ |
| Features with barrel exports | 2/16 | 16/16 | __ |
| ESLint violations | __ | 0 | __ |
| Bundle size increase | Baseline | < 5% | __ |
| Test coverage | __% | Same or better | __ |

### Qualitative Metrics

- [ ] All shared components use facades, not direct feature imports
- [ ] All features expose clean public API via index.ts
- [ ] ESLint enforces layer boundaries
- [ ] No global state hacks (window.__achievementStore removed)
- [ ] Widgets are reusable across content types
- [ ] Developer experience improved (clearer imports, better autocomplete)
- [ ] New contributors can understand architecture from CLAUDE.md

---

## Post-Migration Checklist

### Documentation Updates

- [ ] Update CLAUDE.md with Hybrid Modular architecture
- [ ] Add widgets/ section to architecture docs
- [ ] Document facade pattern in CONTRIBUTING.md
- [ ] Update import examples in README.md
- [ ] Add migration guide for future features

### Code Quality

- [ ] All ESLint rules passing
- [ ] All TypeScript checks passing
- [ ] All tests passing
- [ ] No console warnings in dev mode
- [ ] Bundle size within target

### User-Facing

- [ ] All game modes functional
- [ ] Stats tracking accurate
- [ ] Achievements unlocking correctly
- [ ] Preferences persisting
- [ ] Audio system working
- [ ] No visual regressions

### Deployment

- [ ] Create final commit with migration summary
- [ ] Create PR from feature branch
- [ ] Request code review
- [ ] Merge to main after approval
- [ ] Deploy to staging
- [ ] Monitor for errors
- [ ] Deploy to production
- [ ] Monitor analytics for user impact

---

## Final Commit Message

```bash
git add -A
git commit -m "refactor: migrate to Hybrid Modular architecture

BREAKING CHANGE: Internal architecture restructured for better maintainability

## Summary
Migrated KanaDojo codebase to Hybrid Modular architecture combining
feature-based structure with selective FSD principles.

## Changes

### New Layers
- widgets/ - Complex UI compositions (TrainingGame, MenuWidget, GameUIWidget)
- features/*/facade/ - Public hook APIs for cross-feature access
- shared/events/ - Event bus for decoupled cross-cutting concerns

### Eliminated
- 27 layer violations (shared → features)
- 540 lines of game logic duplication (90% reduction)
- Global state hacks (window.__achievementStore)
- Hub pattern (Progress store imported 25+ times)

### Added
- Barrel exports (index.ts) for all 16 features
- ESLint import rules enforcing layer boundaries
- ContentAdapter pattern for polymorphic game logic
- Event-based stats/achievement system

### Updated
- All shared/ components use facades instead of direct imports
- All app/ pages use barrel exports
- Game flow unified via TrainingGame widget
- Menu system unified via MenuWidget

## Metrics
- Layer violations: 27 → 0 ✅
- Code duplication: 540 lines → <100 lines ✅
- Features with public API: 2/16 → 16/16 ✅
- ESLint violations: 0 ✅

## Testing
- ✅ All automated tests passing
- ✅ Manual testing complete (all game modes)
- ✅ Performance benchmarks met
- ✅ No regressions detected

## Migration Guide
See HYBRID_MODULAR_MIGRATION_PLAN.md for full details.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

---

## Next Steps After Migration

1. **Monitor Production:**
   - Watch error tracking (Sentry/etc.) for facade-related errors
   - Monitor analytics for user behavior changes
   - Check performance metrics (Core Web Vitals)

2. **Documentation:**
   - Write blog post about migration experience
   - Create video walkthrough of new architecture
   - Update onboarding docs for new contributors

3. **Future Features:**
   - Use new architecture as template
   - All new features must include facade/
   - All new features must have index.ts
   - All complex UI goes in widgets/

4. **Continuous Improvement:**
   - Identify remaining code smells
   - Refactor legacy components as needed
   - Add more ESLint rules for stricter enforcement
   - Consider adding Storybook for widget development

---

**END OF MIGRATION PLAN**

Good luck with the migration! Follow this plan phase by phase, and don't hesitate to adjust based on your specific needs. The key is to maintain backwards compatibility during migration and test thoroughly at each checkpoint.
