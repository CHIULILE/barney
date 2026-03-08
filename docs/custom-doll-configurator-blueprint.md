# Custom Doll Configurator Blueprint

這份文件提供一個可直接落地的「高客製化娃娃選配系統」藍圖，重點是：

- 選項多且有相依/互斥規則
- 支援即時計價
- 下單時保留配置快照，避免日後規則改版造成對帳混亂

## 1. Domain Model（核心概念）

- `ProductModel`: 可被客製化的商品主模型（例如某款娃娃系列）
- `OptionGroup`: 選項群組（例如頭部、眼睛、髮型、關節）
- `Option`: 群組內可選項目
- `ConstraintRule`: 規則（requires、excludes、cardinality、conditional_visibility）
- `PriceRule`: 計價規則（固定加價、條件加價、折扣）
- `ConfigurationDraft`: 使用者暫存配置
- `ConfigurationSnapshot`: 下單時的不可變配置快照
- `Order`: 訂單

---

## 2. PostgreSQL Schema（MVP 版）

> 建議先做 MVP，後續再做多語系與更細緻版本管理。

```sql
-- 1) 商品主模型
CREATE TABLE product_models (
  id UUID PRIMARY KEY,
  sku TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  base_price NUMERIC(12,2) NOT NULL,
  currency TEXT NOT NULL DEFAULT 'TWD',
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 2) 選項群組
CREATE TABLE option_groups (
  id UUID PRIMARY KEY,
  product_model_id UUID NOT NULL REFERENCES product_models(id) ON DELETE CASCADE,
  code TEXT NOT NULL,
  name TEXT NOT NULL,
  selection_mode TEXT NOT NULL CHECK (selection_mode IN ('single', 'multiple')),
  min_select INT NOT NULL DEFAULT 0,
  max_select INT,
  sort_order INT NOT NULL DEFAULT 0,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(product_model_id, code)
);

-- 3) 選項
CREATE TABLE options (
  id UUID PRIMARY KEY,
  option_group_id UUID NOT NULL REFERENCES option_groups(id) ON DELETE CASCADE,
  code TEXT NOT NULL,
  name TEXT NOT NULL,
  price_delta NUMERIC(12,2) NOT NULL DEFAULT 0,
  is_default BOOLEAN NOT NULL DEFAULT FALSE,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  sort_order INT NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(option_group_id, code)
);

-- 4) 規則
CREATE TABLE constraint_rules (
  id UUID PRIMARY KEY,
  product_model_id UUID NOT NULL REFERENCES product_models(id) ON DELETE CASCADE,
  rule_type TEXT NOT NULL CHECK (rule_type IN (
    'requires',                -- A requires B
    'excludes',                -- A excludes B
    'cardinality',             -- 群組最少/最多
    'conditional_visibility'   -- 條件顯示
  )),
  priority INT NOT NULL DEFAULT 100,
  condition_json JSONB NOT NULL,
  action_json JSONB NOT NULL,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 5) 計價規則
CREATE TABLE price_rules (
  id UUID PRIMARY KEY,
  product_model_id UUID NOT NULL REFERENCES product_models(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  condition_json JSONB NOT NULL,
  effect_json JSONB NOT NULL,
  priority INT NOT NULL DEFAULT 100,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 6) 配置草稿
CREATE TABLE configuration_drafts (
  id UUID PRIMARY KEY,
  user_id UUID,
  product_model_id UUID NOT NULL REFERENCES product_models(id),
  selected_options JSONB NOT NULL,
  computed_price NUMERIC(12,2),
  currency TEXT,
  validation_result JSONB,
  status TEXT NOT NULL DEFAULT 'editing' CHECK (status IN ('editing', 'validated', 'expired')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 7) 訂單
CREATE TABLE orders (
  id UUID PRIMARY KEY,
  order_no TEXT UNIQUE NOT NULL,
  user_id UUID,
  status TEXT NOT NULL CHECK (status IN ('pending', 'paid', 'in_production', 'shipped', 'cancelled')),
  total_amount NUMERIC(12,2) NOT NULL,
  currency TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 8) 訂單項目 + 配置快照（不可變）
CREATE TABLE order_items (
  id UUID PRIMARY KEY,
  order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_model_id UUID NOT NULL REFERENCES product_models(id),
  qty INT NOT NULL DEFAULT 1,
  unit_price NUMERIC(12,2) NOT NULL,
  config_snapshot JSONB NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 3. Rule JSON 設計範例

### 3.1 requires

```json
{
  "rule_type": "requires",
  "condition": {
    "selected_option_codes": ["JOINT_PRO"]
  },
  "action": {
    "must_include_option_codes": ["TORSO_PRO"]
  }
}
```

### 3.2 excludes

```json
{
  "rule_type": "excludes",
  "condition": {
    "selected_option_codes": ["HEAD_A"]
  },
  "action": {
    "forbid_option_codes": ["NECK_B"]
  }
}
```

### 3.3 cardinality

```json
{
  "rule_type": "cardinality",
  "condition": {
    "option_group_code": "ACCESSORY"
  },
  "action": {
    "min": 0,
    "max": 3
  }
}
```

### 3.4 conditional_visibility

```json
{
  "rule_type": "conditional_visibility",
  "condition": {
    "selected_option_codes": ["STYLE_JP"]
  },
  "action": {
    "show_option_group_codes": ["MAKEUP_JP"]
  }
}
```

---

## 4. API Spec（MVP）

Base URL: `/api/v1`

### 4.1 讀取可配置商品

- `GET /product-models/:id/config-schema`
- 回傳：商品、群組、選項、可見性、預設值、版本資訊

### 4.2 驗證配置

- `POST /configurations/validate`
- Request

```json
{
  "productModelId": "uuid",
  "selectedOptionCodes": ["HEAD_A", "JOINT_PRO", "TORSO_PRO"]
}
```

- Response

```json
{
  "valid": true,
  "errors": [],
  "warnings": [],
  "resolvedState": {
    "disabledOptionCodes": ["NECK_B"],
    "requiredOptionCodes": []
  }
}
```

### 4.3 計價

- `POST /configurations/price`
- Request

```json
{
  "productModelId": "uuid",
  "selectedOptionCodes": ["HEAD_A", "JOINT_PRO", "TORSO_PRO"],
  "quantity": 1
}
```

- Response

```json
{
  "currency": "TWD",
  "basePrice": 12000,
  "optionAdjustments": [
    {"code": "JOINT_PRO", "delta": 1200},
    {"code": "HEAD_A", "delta": 500}
  ],
  "ruleAdjustments": [
    {"name": "PRO_BUNDLE_DISCOUNT", "delta": -300}
  ],
  "total": 13400
}
```

### 4.4 建立配置草稿

- `POST /configuration-drafts`
- 用於「稍後再繼續」和跨裝置

### 4.5 由草稿建立訂單

- `POST /orders/from-draft/:draftId`
- 伺服器端重新驗證 + 重新計價後寫入 `order_items.config_snapshot`

---

## 5. 配置快照（Snapshot）建議內容

下單時必存：

- 商品資料快照（SKU、名稱、基礎價格）
- 選項快照（代碼、名稱、單價增減）
- 規則命中結果（哪些規則影響了結果）
- 最終總價與幣別
- 規則/商品版本號

目標：未來即使你改了選項或規則，也能追溯當時客戶買的是什麼。

---

## 6. 前端互動建議

- 每次使用者更改選配：
  1. 本地先更新選項狀態
  2. 呼叫 `/validate` 拿到錯誤與禁用項
  3. 呼叫 `/price` 顯示最新總價
- UI 必須可解釋：
  - 為什麼這個選項不能選（顯示規則原因）
  - 如果衝突，提供「一鍵修正建議」

---

## 7. MVP 邊界（先別做）

- 不要一開始就做超複雜 rule DSL
- 不要先做 3D 即時預覽
- 不要先接太多金流與物流

先把「可配置 → 驗證 → 計價 → 下單快照」這條路跑通。

---

## 8. 建議技術棧（不綁電商平台）

- Frontend: Next.js + TypeScript
- Backend: NestJS/Fastify (TypeScript)
- DB: PostgreSQL
- Cache（可後加）: Redis

如果團隊偏 Python，也可以用 FastAPI；重點是規則與資料模型設計，不是框架名稱。
