# システム構成詳細設計

## 1. アーキテクチャ概要

```
┌─────────────────────────────────────┐
│           Frontend (SPA)            │
│  ┌─────────────┐ ┌─────────────────┐│
│  │   Vue.js 3  │ │  Audio Module   ││
│  │    Pages    │ │  (Web Speech)   ││
│  └─────────────┘ └─────────────────┘│
│  ┌─────────────┐ ┌─────────────────┐│
│  │    Pinia    │ │   API Client    ││
│  │   (State)   │ │   (AI Service)  ││
│  └─────────────┘ └─────────────────┘│
│  ┌─────────────┐ ┌─────────────────┐│
│  │ LocalStorage│ │   Utilities     ││
│  │  (Persist)  │ │  (Diff/Format)  ││
│  └─────────────┘ └─────────────────┘│
└─────────────────────────────────────┘
              │
              ▼
    ┌─────────────────┐
    │   External APIs │
    │ ┌─────────────┐ │
    │ │   AI APIs   │ │
    │ │ (GPT/Claude)│ │
    │ └─────────────┘ │
    └─────────────────┘
```

## 2. フロントエンド詳細設計

### 2.1 ディレクトリ構造
```
src/
├── components/           # 再利用可能コンポーネント
│   ├── common/          # 共通コンポーネント
│   ├── forms/           # フォーム関連
│   ├── analysis/        # 分析結果表示
│   └── voice/           # 音声入力関連
├── views/               # ページコンポーネント
│   ├── Home.vue        # メイン学習画面
│   ├── History.vue     # 履歴画面
│   └── Settings.vue    # 設定画面
├── stores/              # Pinia ストア
│   ├── learning.js     # 学習データ管理
│   ├── settings.js     # 設定管理
│   └── history.js      # 履歴管理
├── services/            # 外部API連携
│   ├── aiService.js    # AI API クライアント
│   ├── audioService.js # 音声処理
│   └── storageService.js # ローカルストレージ
├── utils/               # ユーティリティ
│   ├── textDiff.js     # テキスト差分処理
│   ├── formatters.js   # データフォーマット
│   └── validators.js   # バリデーション
└── router/              # Vue Router設定
    └── index.js
```

### 2.2 状態管理設計（Pinia）

#### 2.2.1 Learning Store
```javascript
// stores/learning.js
export const useLearningStore = defineStore('learning', {
  state: () => ({
    originalText: '',
    userInputSegments: [], // 分割入力対応: [{id, content, timestamp, type}]
    currentSegment: '',
    analysisResult: null,
    isAnalyzing: false,
    inputMode: 'text', // 'text' | 'voice'
    isRecording: false,
    editingSegmentIndex: -1 // 編集中セグメントのインデックス
  }),
  
  getters: {
    fullUserInput: (state) => state.userInputSegments
      .map(segment => segment.content)
      .join(' '),
    hasInputSegments: (state) => state.userInputSegments.length > 0
  },
  
  actions: {
    setOriginalText(text),
    addInputSegment(content, type = 'text'),
    editInputSegment(index, newContent),
    deleteInputSegment(index),
    moveSegment(fromIndex, toIndex),
    setCurrentSegment(segment),
    startRecording(),
    stopRecording(),
    toggleRecording(),
    clearCurrentSegment(),
    analyzeContent(),
    clearSession()
  }
})
```

#### 2.2.2 Settings Store
```javascript
// stores/settings.js
export const useSettingsStore = defineStore('settings', {
  state: () => ({
    apiProvider: 'openai', // 'openai' | 'anthropic'
    apiKey: '',
    voiceSettings: {
      language: 'ja-JP',
      continuous: false, // 分割入力のため連続認識は無効
      interimResults: true
    }
  }),
  
  persist: true // ローカルストレージに永続化
})
```

#### 2.2.3 History Store
```javascript
// stores/history.js
export const useHistoryStore = defineStore('history', {
  state: () => ({
    sessions: [],
    currentSession: null
  }),
  
  actions: {
    saveSession(sessionData),
    loadHistory(),
    exportToCsv()
  }
})
```

### 2.3 コンポーネント設計

#### 2.3.1 メインコンポーネント構成
- **HomeView**: メイン学習画面
  - TextInputArea: 原文入力エリア
  - **VoiceInputManager**: 音声入力管理コンポーネント
  - **InputSegmentList**: 入力セグメント一覧表示・編集
  - AnalysisResult: 分析結果表示
  - ActionButtons: 実行ボタン群

#### 2.3.2 音声入力関連コンポーネント
- **VoiceInputControls**: 録音開始/停止ボタン
  - 録音中の視覚的フィードバック
  - リアルタイム音声レベル表示
- **InputSegmentItem**: 個別セグメント表示・編集
  - インライン編集機能
  - 削除・並び替えボタン
  - セグメント種別表示（テキスト/音声）
- **SegmentEditor**: セグメント編集モーダル

#### 2.3.3 共通コンポーネント
- **LoadingSpinner**: 読み込み表示
- **ErrorAlert**: エラーメッセージ
- **ConfirmDialog**: 確認ダイアログ
- **DragSortable**: ドラッグ&ドロップ並び替え

## 3. 外部サービス連携

### 3.1 AI API統合
```javascript
// services/aiService.js
class AIService {
  constructor(provider, apiKey) {
    this.provider = provider; // 'openai' | 'anthropic'
    this.apiKey = apiKey;
  }
  
  async analyzeDifference(original, userInput) {
    const prompt = this.buildComparisonPrompt(original, userInput);
    return await this.callAPI(prompt);
  }
}
```

### 3.2 音声認識サービス（拡張版）
```javascript
// services/audioService.js
class AudioService {
  constructor() {
    this.recognition = null;
    this.isRecording = false;
    this.currentSegment = '';
  }
  
  startRecording(onResult, onError) {
    // Web Speech API実装
    // 分割入力対応
    this.recognition.continuous = false;
    this.recognition.interimResults = true;
  }
  
  stopRecording() {
    // 録音停止・セグメント完了
  }
  
  getCurrentSegment() {
    return this.currentSegment;
  }
  
  clearCurrentSegment() {
    this.currentSegment = '';
  }
}
```

## 4. データモデル

### 4.1 学習セッション
```typescript
interface LearningSession {
  id: string;
  timestamp: Date;
  originalText: string;
  inputSegments: InputSegment[];
  analysisResult: AnalysisResult;
  score: number;
}
```

### 4.2 入力セグメント
```typescript
interface InputSegment {
  id: string;
  content: string;
  type: 'text' | 'voice';
  timestamp: Date;
  order: number;
}
```

### 4.3 分析結果
```typescript
interface AnalysisResult {
  differences: DifferenceItem[];
  missing: string[];
  score: number;
  feedback: string;
}

interface DifferenceItem {
  original: string;
  user: string;
  type: 'incorrect' | 'missing' | 'extra';
  position: number;
}
```

## 5. UI/UX設計

### 5.1 音声入力UXフロー
```
1. [録音開始]ボタンをタップ
   ↓
2. マイクアイコンが赤く点滅、音声レベル表示
   ↓
3. 話し終わったら[録音停止]ボタンをタップ
   ↓
4. 認識結果がセグメントとして追加
   ↓
5. 必要に応じて編集・削除・並び替え
   ↓
6. 続けて録音するか、分析実行
```

### 5.2 セグメント管理UI
- セグメント一覧は時系列順に表示
- 各セグメントに編集・削除・移動ボタン
- ドラッグ&ドロップで順序変更可能
- セグメント種別（テキスト/音声）をアイコンで表示

### 5.3 レスポンシブ対応
- モバイル: 縦スタック配置、大きなタップ領域
- タブレット: 2カラム配置
- デスクトップ: 3カラム配置

## 6. 技術仕様

### 6.1 開発環境
- **Node.js**: 20.x
- **Vue.js**: 3.4.x
- **Vite**: 5.x
- **Pinia**: 2.x
- **Vue Router**: 4.x

### 6.2 追加ライブラリ
- **Vue Draggable**: セグメント並び替え
- **Vue Use**: コンポーザブル関数
- **Headless UI**: アクセシブルなUI コンポーネント

### 6.3 テストフレームワーク
- **Unit Test**: Vitest
- **E2E Test**: Playwright
- **Component Test**: Cypress

### 6.4 UI/UXライブラリ
- **CSS Framework**: Tailwind CSS
- **Icons**: Heroicons
- **Animations**: Vue Transition

## 7. セキュリティ考慮事項

### 7.1 API Key管理
- ローカルストレージに暗号化して保存
- 環境変数での管理オプション提供

### 7.2 データ保護
- 学習データはローカルのみ保存
- API通信時のHTTPS必須
- 音声データの一時的な保存のみ

## 8. パフォーマンス最適化

### 8.1 遅延読み込み
- ルートベースのコード分割
- 大きなコンポーネントの動的インポート

### 8.2 状態管理最適化
- セグメント操作の効率化
- 不要な再描画の防止
- computed propertyの活用

### 8.3 音声処理最適化
- 音声認識結果のデバウンス処理
- メモリリーク防止（録音停止時のクリーンアップ）

## 9. 今後の拡張性

### 9.1 バックエンド連携準備
- API抽象化レイヤーの実装
- 認証機能の準備

### 9.2 PWA対応準備
- Service Worker実装準備
- オフライン機能設計

### 9.3 追加機能候補
- セグメント自動区切り機能
- 音声入力時の句読点自動挿入
- キーボードショートカット対応