# 🗃️ Banco de Dados - SevenScale Health Dashboard

Este documento descreve a estrutura completa do banco de dados PostgreSQL (Supabase) utilizado no SevenScale Health Dashboard.

## 📊 Visão Geral

**Plataforma**: Supabase PostgreSQL  
**Schema**: `public`  
**Versão**: 1.0  
**Criado**: 2025-06-25  
**Autor**: Bruno Monteiro - SevenScale  

## 🏗️ Arquitetura do Banco

### Módulos Principais

1. **🤖 AI Agents**: 7 Agentes Impulso® Health + Execuções
2. **👥 Clients**: Clínicas, Métricas e Performance  
3. **🔗 Integrations**: APIs Externas e Sincronização
4. **📋 System**: Logs e Configurações

## 📋 Tabelas Principais

### 🤖 ai_agents
**Descrição**: 7 Agentes Impulso® Health  
**Chave Primária**: `id` (bigint auto-increment)

```sql
CREATE TABLE ai_agents (
  id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  name TEXT NOT NULL,
  agent_type TEXT NOT NULL,
  status TEXT DEFAULT 'active',
  performance INTEGER DEFAULT 0,
  last_execution TIMESTAMPTZ,
  executions_today INTEGER DEFAULT 0,
  success_rate NUMERIC DEFAULT 0.0,
  config JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 📊 agent_executions
**Descrição**: Histórico de execuções dos agentes  
**Chave Primária**: `id` (bigint auto-increment)

```sql
CREATE TABLE agent_executions (
  id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  agent_id BIGINT REFERENCES ai_agents(id),
  client_id BIGINT REFERENCES clients(id),
  execution_date TIMESTAMPTZ DEFAULT NOW(),
  duration_ms INTEGER,
  status TEXT,
  tokens_used INTEGER,
  insights_generated INTEGER,
  execution_details JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 💡 agent_insights  
**Descrição**: Insights gerados pelos agentes  
**Chave Primária**: `id` (uuid auto-generated)

```sql
CREATE TABLE agent_insights (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  client_id BIGINT NOT NULL REFERENCES clients(id),
  agent_type VARCHAR(50) DEFAULT 'mvp-consolidator',
  insights_data JSONB NOT NULL,
  processed_at TIMESTAMPTZ DEFAULT NOW(),
  status VARCHAR(20) DEFAULT 'completed',
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 🏥 clients
**Descrição**: Clínicas e médicos - Tabela central  
**Chave Primária**: `id` (bigint auto-increment)

```sql
CREATE TABLE clients (
  id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  name TEXT NOT NULL,
  specialty TEXT NOT NULL,
  email TEXT NOT NULL,
  phone TEXT,
  city TEXT,
  status TEXT DEFAULT 'operational',
  performance INTEGER DEFAULT 0,
  revenue TEXT,
  patients INTEGER DEFAULT 0,
  plan TEXT DEFAULT 'pro',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 📈 client_metrics
**Descrição**: Métricas específicas por cliente  
**Chave Primária**: `id` (bigint auto-increment)

```sql
CREATE TABLE client_metrics (
  id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  client_id BIGINT REFERENCES clients(id),
  metric_type TEXT NOT NULL,
  metric_value NUMERIC NOT NULL,
  metric_date DATE DEFAULT CURRENT_DATE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 📊 dashboard_metrics
**Descrição**: Métricas consolidadas do dashboard  
**Chave Primária**: `id` (bigint auto-increment)

```sql
CREATE TABLE dashboard_metrics (
  id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  client_id BIGINT REFERENCES clients(id),
  metric_date DATE DEFAULT CURRENT_DATE,
  roi NUMERIC,
  patients_total INTEGER,
  patients_new INTEGER,
  patients_recurring INTEGER,
  revenue NUMERIC,
  appointments_total INTEGER,
  appointments_completed INTEGER,
  appointments_no_show INTEGER,
  leads_total INTEGER,
  leads_qualified INTEGER,
  conversion_rate NUMERIC,
  performance_score INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 🔗 integrations
**Descrição**: Integrações disponíveis por cliente  
**Chave Primária**: `id` (bigint auto-increment)

```sql
CREATE TABLE integrations (
  id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  client_id BIGINT REFERENCES clients(id),
  integration_type TEXT NOT NULL,
  status TEXT DEFAULT 'disconnected',
  last_sync TIMESTAMPTZ,
  config JSONB DEFAULT '{}',
  metrics JSONB DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

### 📋 system_logs
**Descrição**: Logs do sistema  
**Chave Primária**: `id` (bigint auto-increment)

```sql
CREATE TABLE system_logs (
  id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  log_type TEXT NOT NULL,
  message TEXT NOT NULL,
  details JSONB DEFAULT '{}',
  client_id BIGINT REFERENCES clients(id),
  agent_id BIGINT REFERENCES ai_agents(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## 📊 Views Especializadas

### 🎯 client_dashboard_view
**Propósito**: Dashboard consolidado por cliente

```sql
CREATE VIEW client_dashboard_view AS
SELECT 
  c.id,
  c.name AS clinic_name,
  c.specialty,
  c.city,
  c.status,
  c.performance,
  c.revenue,
  c.patients,
  COUNT(ci.id) AS total_integrations,
  COUNT(CASE WHEN ci.status = 'connected' THEN 1 END) AS active_integrations,
  ci.status AS latest_status,
  ai.processed_at AS last_insight_at,
  dm.roi AS roi_percent
FROM clients c
LEFT JOIN client_integrations ci ON c.id = ci.client_id
LEFT JOIN agent_insights ai ON c.id = ai.client_id
LEFT JOIN dashboard_metrics dm ON c.id = dm.client_id
GROUP BY c.id, ci.status, ai.processed_at, dm.roi;
```

### 🤖 vw_agent_analytics
**Propósito**: Analytics dos 7 Agentes Impulso®

```sql
CREATE VIEW vw_agent_analytics AS
SELECT 
  a.id,
  a.name,
  a.agent_type,
  a.status,
  a.performance,
  a.executions_today,
  a.success_rate,
  COUNT(ae.id) AS executions_24h,
  AVG(ae.duration_ms) AS avg_duration_ms,
  COUNT(CASE WHEN ae.status = 'completed' THEN 1 END) AS successful_executions_24h,
  a.last_execution
FROM ai_agents a
LEFT JOIN agent_executions ae ON a.id = ae.agent_id 
  AND ae.execution_date >= NOW() - INTERVAL '24 hours'
GROUP BY a.id;
```

### 📈 vw_dashboard_overview
**Propósito**: KPIs gerais do dashboard

```sql
CREATE VIEW vw_dashboard_overview AS
SELECT 
  COUNT(DISTINCT c.id) AS total_clients,
  COUNT(DISTINCT CASE WHEN c.status = 'operational' THEN c.id END) AS operational_clients,
  COUNT(DISTINCT CASE WHEN c.status = 'attention' THEN c.id END) AS attention_clients,
  COUNT(DISTINCT CASE WHEN c.status = 'critical' THEN c.id END) AS critical_clients,
  COUNT(DISTINCT CASE WHEN a.status = 'active' THEN a.id END) AS active_agents,
  COUNT(DISTINCT a.id) AS total_agents,
  COUNT(DISTINCT CASE WHEN i.status = 'connected' THEN i.id END) AS connected_integrations,
  COUNT(DISTINCT i.id) AS total_integrations,
  SUM(COALESCE(c.revenue::NUMERIC, 0)) AS total_revenue,
  AVG(c.performance) AS avg_client_performance,
  AVG(a.performance) AS avg_agent_performance,
  COUNT(CASE WHEN sl.created_at >= NOW() - INTERVAL '24 hours' THEN 1 END) AS logs_24h,
  NOW() AS generated_at
FROM clients c
CROSS JOIN ai_agents a
CROSS JOIN integrations i
LEFT JOIN system_logs sl ON TRUE;
```

## 🔗 Integrações Suportadas

### Tipos de Integração
- `google_calendar` - Google Calendar API
- `calendly` - Calendly API  
- `hubspot_crm` - HubSpot CRM
- `google_ads` - Google Ads API
- `meta_ads` - Meta Ads API
- `whatsapp_business` - WhatsApp Business API
- `google_analytics` - Google Analytics API

### Fluxo de Dados
1. **Coleta**: `integration_data` armazena dados brutos das APIs
2. **Processamento**: 7 Agentes IA processam os dados
3. **Insights**: `agent_insights` armazena insights processados  
4. **Dashboard**: Views consolidam dados para frontend

## 📊 Métricas Principais

### Financeiras
- `revenue` - Receita
- `roi` - Retorno sobre investimento
- `growth_rate` - Taxa de crescimento
- `conversion_rate` - Taxa de conversão

### Pacientes
- `patients_total` - Total de pacientes
- `patients_new` - Novos pacientes
- `patients_recurring` - Pacientes recorrentes
- `performance_score` - Score de performance

### Marketing
- `leads_total` - Total de leads
- `leads_qualified` - Leads qualificados
- `appointments_total` - Total de agendamentos
- `appointments_completed` - Agendamentos realizados

## 🔒 Segurança (RLS)

### Row Level Security Habilitada
```sql
ALTER TABLE clients ENABLE ROW LEVEL SECURITY;
ALTER TABLE client_metrics ENABLE ROW LEVEL SECURITY;
ALTER TABLE client_integrations ENABLE ROW LEVEL SECURITY;
ALTER TABLE integration_data ENABLE ROW LEVEL SECURITY;
ALTER TABLE agent_insights ENABLE ROW LEVEL SECURITY;
ALTER TABLE dashboard_metrics ENABLE ROW LEVEL SECURITY;
```

### Políticas de Acesso
- **Isolamento por cliente**: Usuários só veem dados dos próprios clientes
- **Autenticação JWT**: Tokens Supabase para validação
- **Campos sensíveis**: `credentials`, `email`, `phone` protegidos

## 🚀 Roadmap do Banco

### Fase 1: Backend (2 semanas)
- [x] Estrutura básica das tabelas
- [x] Views especializadas  
- [x] Políticas RLS
- [ ] FastAPI CRUD endpoints
- [ ] Integração com APIs externas

### Fase 2: Dashboard (2 semanas)
- [x] Métricas em tempo real
- [ ] Relatórios avançados
- [ ] Sistema de alertas
- [ ] Otimização de views

---

**Criado por Bruno Monteiro - SevenScale**  
*Estrutura de dados para transformação digital de clínicas médicas*