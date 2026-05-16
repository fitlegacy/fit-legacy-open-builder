# Roadmap 3-6 Months: Scale Phase

Este documento detalla la implementación de las tres iniciativas clave para el mediano plazo:

1. **Analytics Avanzados** - Trackear engagement de rutinas compartidas
2. **Marketplace de Templates** - Catálogo de rutinas pre-hechas
3. **API Pública** - Permitir que terceros construyan clientes

---

## 1. Analytics Avanzados

### Objetivo
Medir cómo los usuarios interactúan con las rutinas compartidas: quién las abre, cuánto tiempo gastan, cuáles completaron, si las re-compartieron.

### Tabla de Base de Datos: `routine_analytics`

```sql
CREATE TABLE routine_analytics (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  routine_id TEXT NOT NULL,
  wir_hash TEXT NOT NULL UNIQUE,
  creator_id uuid REFERENCES auth.users,
  created_at TIMESTAMP DEFAULT now(),
  
  -- Metadata
  routine_name TEXT,
  routine_type TEXT, -- 'workout', 'nutrition', 'mixed'
  exercises_count INT,
  foods_count INT,
  
  UNIQUE(wir_hash)
);

CREATE TABLE routine_views (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  routine_id TEXT NOT NULL,
  wir_hash TEXT NOT NULL,
  viewed_at TIMESTAMP DEFAULT now(),
  
  -- Metadata del viewer (no requerido login)
  ip_hash TEXT,
  user_agent TEXT,
  referrer TEXT,
  
  -- Engagement
  time_spent_seconds INT,
  completed BOOLEAN DEFAULT false,
  items_checked INT DEFAULT 0,
  total_items INT,
  
  -- Re-share
  reshared_at TIMESTAMP,
  reshare_count INT DEFAULT 0,
  
  FOREIGN KEY (wir_hash) REFERENCES routine_analytics(wir_hash)
);

CREATE INDEX idx_routine_analytics_creator_id ON routine_analytics(creator_id);
CREATE INDEX idx_routine_views_wir_hash ON routine_views(wir_hash);
CREATE INDEX idx_routine_views_viewed_at ON routine_views(viewed_at);
```

### Frontend: Tracking Pixel

En `src/app/components/routine/SharedRoutineViewer.tsx`:

```typescript
// 1. Cuando se carga la rutina
useEffect(() => {
  const trackView = async () => {
    const wirHash = hashWirPayload(wirData);
    
    const response = await fetch('/api/analytics/track-view', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        wir_hash: wirHash,
        routine_name: wirData.n,
        routine_type: wirData.t,
        exercises_count: wirData.e?.length || 0,
        foods_count: wirData.f?.length || 0,
      }),
    });
  };
  
  trackView();
}, [wirData]);

// 2. Cuando el usuario verifica items
const handleItemCheck = async (itemId: string, checked: boolean) => {
  setCompletedItems(prev => 
    checked ? [...prev, itemId] : prev.filter(id => id !== itemId)
  );
  
  // Track completion
  const wirHash = hashWirPayload(wirData);
  await fetch('/api/analytics/track-completion', {
    method: 'POST',
    body: JSON.stringify({
      wir_hash: wirHash,
      items_checked: completedItems.length,
      total_items: (wirData.e?.length || 0) + (wirData.f?.length || 0),
    }),
  });
};

// 3. Cuando el usuario intenta re-compartir
const handleReshare = async () => {
  const wirHash = hashWirPayload(wirData);
  await fetch('/api/analytics/track-reshare', {
    method: 'POST',
    body: JSON.stringify({ wir_hash: wirHash }),
  });
  
  // Share logic...
};
```

### API Backend: Endpoints

En `api/analytics.ts` (Supabase Edge Functions o API Routes):

```typescript
// POST /api/analytics/track-view
export async function trackView(req: Request) {
  const { wir_hash, routine_name, routine_type, exercises_count, foods_count } = await req.json();
  
  const supabase = createClient();
  
  // Crear registro en routine_analytics si no existe
  await supabase
    .from('routine_analytics')
    .upsert(
      {
        wir_hash,
        routine_name,
        routine_type,
        exercises_count,
        foods_count,
      },
      { onConflict: 'wir_hash' }
    );
  
  // Registrar vista
  const { data: { session } } = await supabase.auth.getSession();
  
  await supabase.from('routine_views').insert({
    wir_hash,
    ip_hash: hashIp(req.headers.get('x-forwarded-for')),
    user_agent: req.headers.get('user-agent'),
    referrer: req.headers.get('referer'),
    time_spent_seconds: 0,
    items_checked: 0,
  });
  
  return new Response(JSON.stringify({ ok: true }));
}

// GET /api/analytics/:wir_hash
export async function getAnalytics(req: Request, { wir_hash }: { wir_hash: string }) {
  const supabase = createClient();
  
  const { data: routine } = await supabase
    .from('routine_analytics')
    .select('*')
    .eq('wir_hash', wir_hash)
    .single();
  
  // Solo el creador puede ver analytics
  const { data: { session } } = await supabase.auth.getSession();
  if (routine.creator_id !== session?.user.id) {
    return new Response('Unauthorized', { status: 403 });
  }
  
  const { data: views } = await supabase
    .from('routine_views')
    .select('*')
    .eq('wir_hash', wir_hash)
    .order('viewed_at', { ascending: false });
  
  return new Response(JSON.stringify({
    routine,
    stats: {
      total_views: views.length,
      completion_rate: views.filter(v => v.completed).length / views.length,
      avg_items_checked: views.reduce((sum, v) => sum + v.items_checked, 0) / views.length,
      reshare_count: views.filter(v => v.reshared_at).length,
      avg_time_spent: views.reduce((sum, v) => sum + v.time_spent_seconds, 0) / views.length,
    },
    recent_views: views.slice(0, 20),
  }));
}
```

### Dashboard: Panel de Control

En `src/app/components/analytics/RoutineStatsModal.tsx`:

```typescript
export function RoutineStatsModal({ wirHash }: { wirHash: string }) {
  const [stats, setStats] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch(`/api/analytics/${wirHash}`)
      .then(r => r.json())
      .then(setStats)
      .finally(() => setLoading(false));
  }, [wirHash]);
  
  if (loading) return <div>Loading...</div>;
  
  return (
    <div className="space-y-4">
      <h2>{stats.routine.routine_name}</h2>
      
      <div className="grid grid-cols-2 gap-4">
        <Card>
          <div className="text-3xl font-bold">{stats.stats.total_views}</div>
          <div className="text-sm text-gray-500">Views</div>
        </Card>
        
        <Card>
          <div className="text-3xl font-bold">
            {(stats.stats.completion_rate * 100).toFixed(1)}%
          </div>
          <div className="text-sm text-gray-500">Completion Rate</div>
        </Card>
        
        <Card>
          <div className="text-3xl font-bold">{stats.stats.reshare_count}</div>
          <div className="text-sm text-gray-500">Re-shares</div>
        </Card>
        
        <Card>
          <div className="text-3xl font-bold">
            {Math.floor(stats.stats.avg_time_spent / 60)}m
          </div>
          <div className="text-sm text-gray-500">Avg Time</div>
        </Card>
      </div>
      
      <div>
        <h3 className="font-semibold mb-2">Recent Views</h3>
        <div className="space-y-2">
          {stats.recent_views.map(view => (
            <div key={view.id} className="text-sm border-l-2 border-blue-500 pl-2">
              <div>Viewed: {new Date(view.viewed_at).toLocaleString()}</div>
              <div>{view.items_checked}/{view.total_items} items completed</div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

---

## 2. Marketplace de Templates

### Objetivo
Permitir que coaches compartan sus mejores rutinas para que otros las reutilicen/customicen. Monetizar con comisiones o premium features.

### Tabla de Base de Datos: `marketplace_templates`

```sql
CREATE TABLE marketplace_templates (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  creator_id uuid REFERENCES auth.users,
  
  name TEXT NOT NULL,
  description TEXT,
  category TEXT, -- 'upper_body', 'cardio', 'nutrition', 'mixed', etc
  tags TEXT[], -- Array de tags: ['full_body', 'beginners', 'no_equipment']
  
  wir_payload TEXT NOT NULL, -- El WIR completo
  cover_image_url TEXT,
  
  -- Precio
  price_cents INT DEFAULT 0, -- 0 = gratuito
  
  -- Stats
  downloads INT DEFAULT 0,
  rating FLOAT DEFAULT 0,
  rating_count INT DEFAULT 0,
  
  -- Meta
  published BOOLEAN DEFAULT false,
  published_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now(),
  
  FOREIGN KEY (creator_id) REFERENCES auth.users
);

CREATE TABLE marketplace_downloads (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  template_id uuid REFERENCES marketplace_templates,
  user_id uuid REFERENCES auth.users,
  downloaded_at TIMESTAMP DEFAULT now(),
  
  UNIQUE(template_id, user_id)
);

CREATE TABLE marketplace_ratings (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  template_id uuid REFERENCES marketplace_templates,
  user_id uuid REFERENCES auth.users,
  rating INT CHECK (rating >= 1 AND rating <= 5),
  review TEXT,
  rated_at TIMESTAMP DEFAULT now(),
  
  UNIQUE(template_id, user_id)
);

CREATE INDEX idx_marketplace_templates_category ON marketplace_templates(category);
CREATE INDEX idx_marketplace_templates_published ON marketplace_templates(published);
CREATE INDEX idx_marketplace_templates_rating ON marketplace_templates(rating DESC);
```

### Frontend: Template Creator

En `src/app/components/marketplace/PublishTemplateModal.tsx`:

```typescript
export function PublishTemplateModal({ wirPayload }: { wirPayload: string }) {
  const [formData, setFormData] = useState({
    name: '',
    description: '',
    category: 'mixed',
    tags: [] as string[],
    price: 0,
    coverImageUrl: '',
  });
  
  const handlePublish = async () => {
    const response = await fetch('/api/marketplace/publish', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ...formData,
        wir_payload: wirPayload,
      }),
    });
    
    if (response.ok) {
      toast.success('Template published!');
      // Redirect to marketplace
    }
  };
  
  return (
    <div className="space-y-4">
      <Input
        placeholder="Template Name"
        value={formData.name}
        onChange={e => setFormData({ ...formData, name: e.target.value })}
      />
      
      <Textarea
        placeholder="Description (what's special about this routine?)"
        value={formData.description}
        onChange={e => setFormData({ ...formData, description: e.target.value })}
      />
      
      <Select value={formData.category} onValueChange={category => setFormData({ ...formData, category })}>
        <SelectItem value="upper_body">Upper Body</SelectItem>
        <SelectItem value="lower_body">Lower Body</SelectItem>
        <SelectItem value="full_body">Full Body</SelectItem>
        <SelectItem value="cardio">Cardio</SelectItem>
        <SelectItem value="nutrition">Nutrition Only</SelectItem>
        <SelectItem value="mixed">Mixed Workout + Nutrition</SelectItem>
      </Select>
      
      <div>
        <label className="text-sm font-medium">Price (0 = free)</label>
        <Input
          type="number"
          value={formData.price / 100}
          onChange={e => setFormData({ ...formData, price: parseInt(e.target.value) * 100 })}
          placeholder="0.00"
        />
      </div>
      
      <TagInput
        value={formData.tags}
        onChange={tags => setFormData({ ...formData, tags })}
        suggestions={['full_body', 'beginners', 'advanced', 'no_equipment', 'home', 'gym']}
      />
      
      <Button onClick={handlePublish} className="w-full">
        Publish Template
      </Button>
    </div>
  );
}
```

### Frontend: Marketplace Browse

En `src/app/components/marketplace/TemplateGallery.tsx`:

```typescript
export function TemplateGallery() {
  const [templates, setTemplates] = useState<Template[]>([]);
  const [filters, setFilters] = useState({ category: '', search: '' });
  
  useEffect(() => {
    const query = new URLSearchParams({
      category: filters.category,
      search: filters.search,
    });
    
    fetch(`/api/marketplace/search?${query}`)
      .then(r => r.json())
      .then(setTemplates);
  }, [filters]);
  
  return (
    <div>
      <div className="flex gap-4 mb-6">
        <Input
          placeholder="Search templates..."
          onChange={e => setFilters({ ...filters, search: e.target.value })}
        />
        
        <Select value={filters.category} onValueChange={category => setFilters({ ...filters, category })}>
          <SelectItem value="">All Categories</SelectItem>
          <SelectItem value="upper_body">Upper Body</SelectItem>
          <SelectItem value="lower_body">Lower Body</SelectItem>
          {/* ... */}
        </Select>
      </div>
      
      <div className="grid grid-cols-3 gap-4">
        {templates.map(template => (
          <TemplateCard key={template.id} template={template} />
        ))}
      </div>
    </div>
  );
}

function TemplateCard({ template }: { template: Template }) {
  const handleUseTemplate = async () => {
    const response = await fetch(`/api/marketplace/download/${template.id}`, {
      method: 'POST',
    });
    
    if (response.ok) {
      const { wir_payload } = await response.json();
      // Cargar en el builder
      useRoutineStore.setState({ wirPayload: wir_payload });
      // Navigate to builder
    }
  };
  
  return (
    <div className="border rounded-lg p-4">
      {template.cover_image_url && (
        <img src={template.cover_image_url} alt={template.name} className="w-full h-40 object-cover rounded" />
      )}
      
      <h3 className="font-semibold mt-2">{template.name}</h3>
      <p className="text-sm text-gray-600">{template.description}</p>
      
      <div className="flex gap-2 mt-2">
        <Badge>{template.category}</Badge>
        {template.tags.slice(0, 2).map(tag => (
          <Badge key={tag} variant="outline">{tag}</Badge>
        ))}
      </div>
      
      <div className="flex items-center justify-between mt-4">
        <div>
          <div className="flex gap-1">
            {Array(5).fill(0).map((_, i) => (
              <Star key={i} size={14} fill={i < Math.round(template.rating) ? 'currentColor' : 'none'} />
            ))}
          </div>
          <span className="text-xs text-gray-500">({template.rating_count})</span>
        </div>
        
        <span className="text-lg font-bold">
          {template.price > 0 ? `$${(template.price / 100).toFixed(2)}` : 'Free'}
        </span>
      </div>
      
      <Button onClick={handleUseTemplate} className="w-full mt-4">
        Use Template
      </Button>
    </div>
  );
}
```

### API Backend: Marketplace Endpoints

En `api/marketplace.ts`:

```typescript
// POST /api/marketplace/publish
export async function publishTemplate(req: Request) {
  const { name, description, category, tags, price, wir_payload, coverImageUrl } = await req.json();
  const supabase = createClient();
  const { data: { session } } = await supabase.auth.getSession();
  
  if (!session) return new Response('Unauthorized', { status: 401 });
  
  const { data, error } = await supabase
    .from('marketplace_templates')
    .insert({
      creator_id: session.user.id,
      name,
      description,
      category,
      tags,
      price,
      wir_payload,
      cover_image_url: coverImageUrl,
      published: true,
      published_at: new Date(),
    })
    .select()
    .single();
  
  return new Response(JSON.stringify(data));
}

// GET /api/marketplace/search?category=&search=&sort=rating
export async function searchTemplates(req: Request) {
  const url = new URL(req.url);
  const category = url.searchParams.get('category');
  const search = url.searchParams.get('search') || '';
  const sort = url.searchParams.get('sort') || 'rating';
  
  const supabase = createClient();
  
  let query = supabase
    .from('marketplace_templates')
    .select('*')
    .eq('published', true);
  
  if (category) query = query.eq('category', category);
  if (search) query = query.or(`name.ilike.%${search}%,description.ilike.%${search}%`);
  
  const { data } = await query.order(sort, { ascending: false });
  
  return new Response(JSON.stringify(data));
}

// POST /api/marketplace/download/:templateId
export async function downloadTemplate(req: Request, { templateId }: { templateId: string }) {
  const supabase = createClient();
  const { data: { session } } = await supabase.auth.getSession();
  
  if (!session) return new Response('Unauthorized', { status: 401 });
  
  // Registrar descarga
  await supabase
    .from('marketplace_downloads')
    .insert({
      template_id: templateId,
      user_id: session.user.id,
    })
    .onConflict('template_id, user_id');
  
  // Incrementar contador
  await supabase.rpc('increment_downloads', { template_id: templateId });
  
  // Obtener template
  const { data: template } = await supabase
    .from('marketplace_templates')
    .select('wir_payload')
    .eq('id', templateId)
    .single();
  
  return new Response(JSON.stringify(template));
}

// POST /api/marketplace/rate/:templateId
export async function rateTemplate(req: Request, { templateId }: { templateId: string }) {
  const { rating, review } = await req.json();
  const supabase = createClient();
  const { data: { session } } = await supabase.auth.getSession();
  
  if (!session) return new Response('Unauthorized', { status: 401 });
  
  await supabase
    .from('marketplace_ratings')
    .upsert({
      template_id: templateId,
      user_id: session.user.id,
      rating,
      review,
    });
  
  // Recalcular rating promedio
  const { data: ratings } = await supabase
    .from('marketplace_ratings')
    .select('rating')
    .eq('template_id', templateId);
  
  const avgRating = ratings.reduce((sum, r) => sum + r.rating, 0) / ratings.length;
  
  await supabase
    .from('marketplace_templates')
    .update({ rating: avgRating, rating_count: ratings.length })
    .eq('id', templateId);
  
  return new Response(JSON.stringify({ ok: true }));
}
```

---

## 3. API Pública para Desarrolladores

### Objetivo
Permitir que terceros (app fitness, coachs, integradores) construyan clientes personalizados que lean/escriban rutinas en formato `.wir`.

### API Endpoints

```
GET    /api/public/v1/wir/validate           - Validar un payload WIR
POST   /api/public/v1/wir/encode             - Encodear una rutina a WIR
GET    /api/public/v1/wir/decode/:payload    - Decodificar un WIR
POST   /api/public/v1/routines/create        - Crear rutina (requiere API key)
GET    /api/public/v1/routines/:routineId    - Obtener rutina
GET    /api/public/v1/catalog/exercises      - Listar ejercicios disponibles
GET    /api/public/v1/catalog/foods          - Listar comidas disponibles
GET    /api/public/v1/marketplace/templates  - Buscar templates
POST   /api/public/v1/webhooks               - Registrar webhook para eventos
```

### Autenticación: API Keys

En `supabase/migrations/api_keys.sql`:

```sql
CREATE TABLE api_keys (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid REFERENCES auth.users,
  key_hash TEXT NOT NULL UNIQUE,
  name TEXT,
  
  -- Permisos
  scopes TEXT[], -- ['read_catalog', 'write_routines', 'read_templates']
  
  -- Rate limiting
  rate_limit INT DEFAULT 1000, -- requests per hour
  
  -- Meta
  created_at TIMESTAMP DEFAULT now(),
  last_used_at TIMESTAMP,
  revoked BOOLEAN DEFAULT false
);

CREATE TABLE api_usage (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  api_key_id uuid REFERENCES api_keys,
  endpoint TEXT,
  status_code INT,
  used_at TIMESTAMP DEFAULT now()
);

CREATE INDEX idx_api_usage_api_key_id_used_at ON api_usage(api_key_id, used_at);
```

### SDK/Documentation

Crear `docs/API.md`:

```markdown
# Fit Legacy Public API

## Authentication

All requests require an API key:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" https://api.fitlegacy.app/v1/...
```

## WIR Validation

Check if a WIR payload is valid:

```typescript
import { fitLegacyClient } from '@fit-legacy/sdk';

const client = fitLegacyClient({ apiKey: 'your-key' });

const validation = await client.wir.validate({
  v: 1,
  t: 'workout',
  n: 'My Routine',
  e: [
    { i: 'press_banca', s: 4, r: 10, w: 60 }
  ]
});

if (validation.valid) {
  console.log('Routine is valid!');
  const link = client.wir.generateLink(wirPayload);
} else {
  console.log('Errors:', validation.errors);
}
```

## Catalog Search

Get available exercises or foods:

```typescript
const exercises = await client.catalog.exercises({
  search: 'press',
  limit: 10
});

// Response:
// {
//   data: [
//     { id: 'press_banca', name: 'Barbell Bench Press', category: 'chest', ... }
//   ]
// }
```

## Create Routine (Programmatic)

Create a routine directly in your app and get a shareable link:

```typescript
const routine = await client.routines.create({
  name: 'My Custom Routine',
  type: 'workout',
  exercises: [
    { exerciseId: 'press_banca', sets: 4, reps: 10, weight: 60 },
    { exerciseId: 'pull_ups', sets: 4, reps: 8, weight: 0 }
  ]
});

// Response: { id: 'routine_123', shareLink: 'https://builder.fitlegacy.app/r/wir?data=...' }
```

## Webhooks

Get notified when routines are viewed/completed:

```typescript
await client.webhooks.register({
  url: 'https://your-app.com/webhooks/fit-legacy',
  events: ['routine.viewed', 'routine.completed', 'routine.reshared']
});
```

Webhook payload:

```json
{
  "event": "routine.completed",
  "timestamp": "2026-05-16T22:30:00Z",
  "data": {
    "routine_id": "routine_123",
    "wir_hash": "abc123...",
    "items_checked": 8,
    "total_items": 10,
    "time_spent_seconds": 1200
  }
}
```
```

### SDK NPM Package

En `packages/sdk/`:

```typescript
// lib/index.ts
import { WirClient } from './wir';
import { CatalogClient } from './catalog';
import { RoutinesClient } from './routines';
import { WebhooksClient } from './webhooks';

export interface FitLegacyClientOptions {
  apiKey: string;
  baseUrl?: string;
}

export class FitLegacyClient {
  wir: WirClient;
  catalog: CatalogClient;
  routines: RoutinesClient;
  webhooks: WebhooksClient;
  
  constructor(options: FitLegacyClientOptions) {
    const baseUrl = options.baseUrl || 'https://api.fitlegacy.app';
    
    this.wir = new WirClient(baseUrl, options.apiKey);
    this.catalog = new CatalogClient(baseUrl, options.apiKey);
    this.routines = new RoutinesClient(baseUrl, options.apiKey);
    this.webhooks = new WebhooksClient(baseUrl, options.apiKey);
  }
}

export { default as fitLegacyClient } from './client';
```

En `package.json` del SDK:

```json
{
  "name": "@fit-legacy/sdk",
  "version": "1.0.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "test": "vitest"
  }
}
```

### API Endpoint Implementation

En `api/public/v1/wir/validate.ts`:

```typescript
export async function validateWir(req: Request) {
  const apiKey = req.headers.get('authorization')?.replace('Bearer ', '');
  
  if (!apiKey) {
    return new Response(JSON.stringify({ error: 'Missing API key' }), { status: 401 });
  }
  
  // Validate API key
  const supabase = createClient();
  const keyHash = hashApiKey(apiKey);
  
  const { data: keyRecord, error } = await supabase
    .from('api_keys')
    .select('*')
    .eq('key_hash', keyHash)
    .eq('revoked', false)
    .single();
  
  if (error || !keyRecord) {
    return new Response(JSON.stringify({ error: 'Invalid API key' }), { status: 401 });
  }
  
  // Check scopes
  if (!keyRecord.scopes.includes('read_catalog')) {
    return new Response(JSON.stringify({ error: 'Insufficient permissions' }), { status: 403 });
  }
  
  const wirData = await req.json();
  
  // Import validation logic from main app
  import { validateWir as validate } from '../../../src/lib/wir';
  const result = validate(wirData, { checkCatalog: true });
  
  // Log usage
  await supabase.from('api_usage').insert({
    api_key_id: keyRecord.id,
    endpoint: '/wir/validate',
    status_code: result.valid ? 200 : 400,
  });
  
  return new Response(JSON.stringify(result));
}
```

---

## Cronograma de Implementación (3-6 meses)

### Mes 1-2: Fundación
- [ ] Diseñar schema de BD para analytics
- [ ] Implementar tracking pixel en SharedRoutineViewer
- [ ] Dashboard básico de stats
- [ ] Configurar API key infrastructure

### Mes 2-3: Marketplace MVP
- [ ] Tabla de templates en BD
- [ ] UI de publicación de templates
- [ ] UI de galería de templates
- [ ] Sistema de descargas/ratings

### Mes 3-4: API Pública Beta
- [ ] SDK NPM inicial
- [ ] Endpoints básicos del WIR
- [ ] Documentación API
- [ ] Rate limiting y monitoring

### Mes 4-5: Polish & Scale
- [ ] Analytics avanzados (cohort analysis, retention)
- [ ] Marketplace monetization (Stripe integration)
- [ ] Webhooks en producción
- [ ] Load testing

### Mes 5-6: Promoción
- [ ] Lanzar API públicamente
- [ ] Outreach a app makers
- [ ] Case studies de templates
- [ ] Community integrations

---

## Stack Propuesto

| Funcionalidad | Tech |
|---|---|
| **BD** | Supabase (PostgreSQL) |
| **API** | Vercel Functions / Edge Functions |
| **SDK** | TypeScript + zod |
| **Docs** | Nextra / VitePress |
| **Webhooks** | Svix (o self-hosted) |
| **Monitoring** | Vercel Analytics + Sentry |

---

## Métricas de Éxito

### Analytics
- ✅ 80% de las rutinas compartidas generan al menos 1 view
- ✅ 30% de completion rate promedio

### Marketplace
- ✅ 100+ templates publicados en mes 3
- ✅ 1000+ descargas de templates en mes 4

### API
- ✅ 50+ desarrolladores registrados
- ✅ 100k requests/mes en mes 5
- ✅ 5+ integraciones de terceros

