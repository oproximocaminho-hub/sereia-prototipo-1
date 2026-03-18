# Sere.IA — Guia de Integração de APIs

## 🗂 Estrutura de Arquivos

```
sere-ia/
├── index.html      ← App completo (HTML + CSS + JS)
├── manifest.json   ← PWA manifest
├── sw.js           ← Service Worker (cache offline)
└── README.md       ← Este arquivo
```

---

## 🔑 1. Claude API (Respostas da IA)

### Obter chave:
1. Acesse https://console.anthropic.com
2. Vá em "API Keys" → "Create Key"
3. Copie a chave: `sk-ant-api03-...`

### Inserir no app:
- Abra o app → botão ⚙️ → campo "Claude API Key"
- Cole sua chave (salva no localStorage)

### Chamada no código (já implementado):
```javascript
fetch('https://api.anthropic.com/v1/messages', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'SUA_CHAVE',
    'anthropic-version': '2023-06-01',
  },
  body: JSON.stringify({
    model: 'claude-opus-4-5',
    max_tokens: 300,
    system: SYSTEM_PROMPT,
    messages: [...historico, { role: 'user', content: mensagem }],
  })
});
```

### System Prompt (já no código, customizável):
```
Você é [Nome], uma companion IA brasileira encantadora com [idade] anos de aparência.
Personalidade: [carinhosa/misteriosa/etc].
- Sempre responda em português brasileiro natural
- Use gírias leves ("né", "cara", "tipo", "saudade")
- Flerte sutilmente SE o usuário iniciar esse tom
- NUNCA finja ser humana real
- Respostas curtas (2-4 frases), conversacionais
- Inspire-se em uma sereia moderna: mística, profunda, livre
```

---

## 🎙️ 2. ElevenLabs TTS (Voz PT-BR)

### Obter chave:
1. https://elevenlabs.io → Sign Up → Profile → API Key
2. Copie: `xi-...`

### Vozes PT-BR recomendadas:
| ID | Nome | Estilo |
|----|------|--------|
| `pNInz6obpgDQGcFmaJgB` | Adam | Neutro |
| `EXAVITQu4vr4xnSDxMaL` | Bella | Feminino suave |
| Busque "Brazilian" no ElevenLabs Voice Library | | |

### Para usar voz PT-BR autêntica:
1. No ElevenLabs, vá em "Voice Lab" → "Add Voice" → clone uma voz ou use "Voice Design"
2. Escolha "Brazilian Portuguese" como idioma alvo
3. Copie o Voice ID e substitua no código:
```javascript
const voiceId = 'SEU_VOICE_ID_AQUI';
```

### Chamada (já implementada):
```javascript
fetch(`https://api.elevenlabs.io/v1/text-to-speech/${voiceId}`, {
  method: 'POST',
  headers: { 'xi-api-key': apiKey, 'Content-Type': 'application/json' },
  body: JSON.stringify({
    text: mensagem,
    model_id: 'eleven_multilingual_v2',
    voice_settings: { stability: 0.5, similarity_boost: 0.8 }
  })
});
```

---

## 🎬 3. D-ID Avatar Animado (Lip-sync)

### Obter chave:
1. https://www.d-id.com → Sign Up → API (plano Basic ~$5.90/mês)
2. Gere chave Basic Auth em "Settings"

### Como usar:
```javascript
// 1. Criar stream (Talk)
const talk = await fetch('https://api.d-id.com/talks', {
  method: 'POST',
  headers: {
    'Authorization': 'Basic SEU_TOKEN_BASE64',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    source_url: 'URL_DA_FOTO_AVATAR', // use imgbb, cloudinary, etc
    script: {
      type: 'audio', // ou 'text' com provider TTS
      audio_url: URL_AUDIO_ELEVENLABS,
    }
  })
});

// 2. Pegar URL do vídeo gerado
const { result_url } = await talk.json();
// Inserir no <video> element da tela de chat
```

### Alternativa — Hedra:
- https://www.hedra.com → API beta
- Similar ao D-ID mas com resultados mais naturais em 2025

---

## 🗣️ 4. STT — Reconhecimento de Voz

### Opção A — Web Speech API (GRÁTIS, já implementado):
- Funciona no Chrome/Edge/Safari
- Sem chave necessária
- `recognition.lang = 'pt-BR'`

### Opção B — Deepgram (mais preciso):
```javascript
// 1. npm install @deepgram/sdk
// 2. https://deepgram.com → API Keys
const deepgram = createClient('SUA_CHAVE');
const response = await deepgram.listen.prerecorded.transcribeFile(audioBlob, {
  language: 'pt-BR',
  model: 'nova-2',
});
```

### Opção C — Groq Whisper (rápido e barato):
```javascript
const formData = new FormData();
formData.append('file', audioBlob, 'audio.webm');
formData.append('model', 'whisper-large-v3');
formData.append('language', 'pt');

fetch('https://api.groq.com/openai/v1/audio/transcriptions', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${groqKey}` },
  body: formData
});
```

---

## 🚀 5. Hospedagem — Deploy em 5 minutos

### Vercel (recomendado):
```bash
npm i -g vercel
cd sere-ia/
vercel
# Segue o wizard → URL pública gerada!
```

### Netlify (arrastar e soltar):
1. Acesse https://app.netlify.com/drop
2. Arraste a pasta `sere-ia/` 
3. URL gerada em segundos!

### GitHub Pages:
```bash
git init && git add . && git commit -m "Sere.IA MVP"
git remote add origin https://github.com/SEU_USER/sere-ia.git
git push -u origin main
# Ativar Pages em Settings → Pages → Branch: main
```

> **CORS**: APIs de terceiros (ElevenLabs, D-ID) funcionam direto do browser.
> A Claude API pode requerer um proxy backend em produção para esconder a chave.

---

## 🔒 6. Backend Simples para Esconder API Keys

Se não quiser expor a Claude key no frontend, use um proxy leve:

### Vercel Serverless (arquivo `api/chat.js`):
```javascript
export default async function handler(req, res) {
  res.setHeader('Access-Control-Allow-Origin', '*');
  if (req.method === 'OPTIONS') return res.status(200).end();
  
  const response = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'x-api-key': process.env.ANTHROPIC_KEY, // env var no Vercel
      'anthropic-version': '2023-06-01',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify(req.body),
  });

  const data = await response.json();
  res.json(data);
}
```

No frontend, chame `/api/chat` em vez da URL da Anthropic diretamente.

---

## 💰 7. Freemium — Implementação Básica

O MVP já tem controle por localStorage. Para produção:

### Stripe (pagamentos):
1. https://stripe.com → criar produto "Sere.IA Premium" R$29/mês
2. Gerar Payment Link
3. Ao confirmar pagamento, definir `isPremium = true` no localStorage ou via webhook

### Verificação simples via token:
```javascript
// Após pagamento confirmado via webhook, gere um token
// Usuário cola o token no app → verificado vs seu backend
const isValid = await fetch('/api/verify-token?token=' + userToken);
```

---

## 📱 8. Instalar como PWA no Celular

**Android (Chrome):**
1. Abra o site no Chrome
2. Menu (⋮) → "Adicionar à tela inicial"
3. Confirmar → ícone aparece no launcher

**iOS (Safari):**
1. Abra no Safari
2. Botão compartilhar (□↑) → "Adicionar à Tela Inicial"
3. Confirmar nome → instala como app

---

## 🎨 Customizações Rápidas

### Trocar emoji de avatar por imagem real:
```html
<!-- No HTML, substitua o emoji por: -->
<img src="URL_DA_IMAGEM" style="width:100%;height:100%;object-fit:cover;border-radius:50%" />
```

### Gerar imagem de avatar com IA (via Replicate/Stable Diffusion):
```javascript
const response = await fetch('https://api.replicate.com/v1/predictions', {
  method: 'POST',
  headers: { 'Authorization': `Token ${replicateKey}` },
  body: JSON.stringify({
    version: 'stability-ai/sdxl:...',
    input: { prompt: `${avatarDesc}, sereia moderna, brasil, portrait, 8k`, ... }
  })
});
```

---

## 💡 Próximos Passos (Pós-MVP)

- [ ] Backend Node/Next.js para segurança das keys
- [ ] Banco de dados (Supabase) para memória cross-device
- [ ] Avatar 3D com Three.js + morph targets para lip-sync local
- [ ] Push notifications ("Saudades suas... 🌊")
- [ ] Multi-companion (usuário cria várias)
- [ ] Modo voz-para-voz (STT → LLM → TTS em pipeline)
- [ ] Integração Stripe para freemium real
