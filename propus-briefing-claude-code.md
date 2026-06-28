# Propus — Briefing completo do projeto

Você vai me ajudar a construir o **Propus**, um SaaS de gerador de orçamentos profissionais
para autônomos e MEIs brasileiros. Abaixo está todo o contexto do produto, decisões já tomadas,
identidade visual e o que precisa ser construído. Leia tudo antes de começar.

---

## 1. O produto

**Propus** é um SaaS low-ticket de gerador de orçamentos profissionais voltado para autônomos
e MEIs brasileiros (pintores, fotógrafos, marceneiros, eletricistas, técnicos de TI, designers,
personal trainers, etc.).

### Problema que resolve
A maioria dos autônomos passa orçamento digitado no WhatsApp ou em texto corrido. Isso transmite
amadorismo, não gera histórico, e o prestador não sabe se o cliente abriu ou ignorou.

### Solução
Plataforma web simples onde o autônomo cria um orçamento em minutos, gera um link profissional
e envia para o cliente. O cliente abre no celular, visualiza o PDF bonito e pode aceitar com
um clique. O prestador recebe notificação quando o cliente abre ou aceita.

### Diferenciais principais
- **Chat de IA**: usuário descreve o orçamento em texto livre ("pintura de sala, mão de obra
  R$500, 2 galões de tinta R$180, validade 7 dias") e a IA preenche todos os campos automaticamente.
- **Templates por segmento**: layouts prontos para cada nicho, personalizáveis com logo e cor da marca.
- **Link de aceite**: cliente aceita o orçamento com um clique, sem precisar baixar nada.
- **Notificação em tempo real**: prestador sabe exatamente quando o cliente abriu.

---

## 2. Identidade visual

| Item | Valor |
|------|-------|
| Nome | Propus |
| Domínio | propus.app.br |
| Cor principal | Navy `#1A3560` |
| Fonte (logo) | Paths vetorizados (sem dependência de fonte) |
| Tom de comunicação | Direto, coloquial, brasileiro — fala como o autônomo fala |

### Arquivos de logo disponíveis (pasta `exports/`)
- `propus-icon.svg` / `propus-icon.png` — ícone quadrado 512×512, fundo transparente
- `propus-icon-white.svg` / `propus-icon-white.png` — versão branca para fundos escuros
- `propus-horizontal.svg` / `propus-horizontal.png` — logo horizontal 1322px
- `propus-inverted.svg` / `propus-inverted.png` — logo invertida (branca, para fundo navy)

O ícone é um documento com uma seta de envio integrada no canto inferior direito.
Navy `#1A3560` em toda a identidade.

---

## 3. Planos e precificação

| Plano | Preço | Limite |
|-------|-------|--------|
| Grátis | R$0 | 3 orçamentos/mês + marca d'água no PDF |
| Pro | R$39/mês | Ilimitado, logo própria, sem marca d'água, WhatsApp, status em tempo real |
| Business | R$79/mês | Tudo do Pro + templates por segmento, follow-up automático, até 3 usuários |

Plano anual: 2 meses grátis (R$390/ano Pro, R$790/ano Business).

---

## 4. MVP — Fase 1 (construir agora)

Estas são as únicas funcionalidades do MVP. Nada além disso na Fase 1.

1. **Cadastro e autenticação** — email/senha + Google OAuth
2. **Onboarding da empresa** — logo, nome, CNPJ, contato (salvo uma vez, usado em todos os PDFs)
3. **Editor de orçamento**
   - Campos: cliente, descrição do serviço, itens (descrição + quantidade + valor unitário)
   - Cálculo automático de total e desconto opcional
   - Prazo de validade, condições de pagamento, observações
4. **Chat de IA** — usuário descreve em texto livre, IA extrai e preenche os campos (API Anthropic)
5. **Geração de PDF profissional** — 3 templates visuais (Clássico, Moderno, Minimalista)
   com cor da marca personalizável
6. **Link de compartilhamento** — URL pública do orçamento, visualização no mobile sem login
7. **Aceite online** — botão "Aceitar orçamento" na página pública
8. **Status em tempo real** — Pendente → Visualizado → Aceito → Recusado
9. **Histórico de orçamentos** — listagem com busca por cliente e filtro por status
10. **Controle de limite freemium** — bloqueia criação após 3/mês e exibe upsell

### Fora do MVP (Fase 2+)
- Envio direto por WhatsApp (deep link)
- Follow-up automático
- Dashboard de conversão e métricas
- Múltiplos usuários / equipe
- Assinatura digital com validade jurídica
- Integração com ERP / Nota Fiscal

---

## 5. Stack recomendada

| Camada | Tecnologia |
|--------|-----------|
| Frontend | Next.js 14 (App Router) + TypeScript |
| Estilização | Tailwind CSS |
| Backend / BaaS | Supabase (auth + postgres + storage) |
| Geração de PDF | `@react-pdf/renderer` ou `pdf-lib` |
| IA (chat) | Anthropic API (`claude-sonnet-4-6`) |
| Pagamentos | Stripe (ou Hotmart como alternativa BR) |
| Deploy | Vercel |
| Email transacional | Resend |

### Variáveis de ambiente necessárias
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
ANTHROPIC_API_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
RESEND_API_KEY=
```

---

## 6. Estrutura de banco de dados (Supabase / Postgres)

```sql
-- Usuários (extende auth.users do Supabase)
profiles (
  id uuid references auth.users primary key,
  company_name text,
  cnpj text,
  email text,
  phone text,
  logo_url text,
  brand_color text default '#1A3560',
  plan text default 'free', -- free | pro | business
  monthly_quote_count int default 0,
  count_reset_at timestamptz,
  created_at timestamptz default now()
)

-- Orçamentos
quotes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references profiles(id),
  number text, -- ex: "047"
  client_name text,
  service_description text,
  items jsonb, -- [{descricao, quantidade, valorUnitario}]
  discount numeric default 0,
  payment_conditions text,
  validity_days int,
  observations text,
  template text default 'classic', -- classic | modern | minimal
  status text default 'pending', -- pending | viewed | accepted | refused
  public_token text unique default gen_random_uuid()::text,
  viewed_at timestamptz,
  accepted_at timestamptz,
  created_at timestamptz default now()
)
```

---

## 7. Fluxo do chat de IA

O endpoint `/api/ai/parse-quote` recebe o texto do usuário e retorna JSON estruturado.

```typescript
// Prompt do sistema
const systemPrompt = `Você extrai dados de orçamentos descritos em texto livre.
Retorne SOMENTE JSON válido, sem markdown, sem texto extra.
Formato:
{
  "cliente": "string ou null",
  "servico": "resumo curto do serviço",
  "itens": [{"descricao": "string", "quantidade": number, "valorUnitario": number}],
  "condicoes": "string ou null",
  "validade": "número de dias como string ou null",
  "observacoes": "string ou null"
}`;
```

**Importante**: a chave da Anthropic fica SOMENTE no backend (nunca exposta no frontend).

---

## 8. Templates de PDF — referência visual

Todos os templates compartilham os mesmos dados, só o layout muda:

- **Clássico**: fundo branco, header com logo + número do orçamento, borda inferior colorida com
  a cor da marca, tabela de itens limpa, botão de aceite no final.
- **Moderno**: header com fundo na cor da marca (navy por padrão), corpo branco com card interno,
  tipografia mais bold.
- **Minimalista**: fundo off-white (#F9F9F7), acento colorido sutil (3px barra), total destacado
  no topo à direita, sem header elaborado.

---

## 9. Landing page de validação

Já existe uma landing page HTML completa em `orcapro-landing.html` com:
- Hero com comparativo "antes/depois" (WhatsApp vs PDF profissional)
- Seção de nichos atendidos
- Problema + solução em 3 passos
- Features listadas
- CTA com captura de e-mail (lista de espera)
- Design responsivo com Google Fonts (Sora + Inter)

**Pendente**: conectar o formulário a um coletor de e-mails (Brevo, Mailchimp ou Tally.so)
e subir na Vercel com o domínio `propus.app.br`.

---

## 10. Próximos passos em ordem

1. Inicializar projeto Next.js + Supabase + Tailwind
2. Configurar autenticação (Supabase Auth com Google OAuth)
3. Criar tabelas no banco (profiles + quotes)
4. Construir onboarding de empresa (logo, dados, cor da marca)
5. Construir editor de orçamento com chat de IA
6. Implementar geração de PDF (3 templates)
7. Implementar página pública do orçamento (link de compartilhamento + aceite)
8. Sistema de notificação de status (email via Resend)
9. Implementar controle freemium (limite de 3/mês)
10. Integrar Stripe para upgrade de plano
11. Subir landing page na Vercel + conectar domínio

---

## 11. Público-alvo e tom de voz

**Quem usa**: autônomos brasileiros que nunca usaram ferramenta de orçamento — pintores,
fotógrafos, marceneiros, técnicos de TI, eletricistas, designers, personal trainers.

**Tom de voz**: direto, sem jargão técnico, coloquial. "Chega de passar orçamento no WhatsApp
digitado." Fala como o autônomo fala, não como uma empresa de software.

**Dor central**: orçamento feito no WhatsApp parece amador, cliente some sem responder,
prestador não sabe se foi lido.

---

Pode começar pelo passo 1: inicializar o projeto e configurar o ambiente.
