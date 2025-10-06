// PROJECT: multi-personality-chat-site
// Files are included below. Save each section to the path shown after the header line.

// --- file: README.md ---
# Multi-Personality Chatbot Website (React + Serverless)

This project is a full website that lets visitors choose among five AI personalities and chat with them using the OpenAI API. The frontend is a single-page React app and the backend is a serverless function (for Vercel/Netlify) that proxies requests to OpenAI so your API key stays secret.

Features
- Homepage with buttons for each personality
- Chat page per personality (real responses from OpenAI)
- Safety filter (client + server) to block dangerous requests
- Dark & futuristic UI

Deployment
- Recommended: Vercel (easy serverless functions + environment vars)
- Alternatives: Netlify (Netlify Functions), Cloudflare Workers, or any server that supports Node.js serverless functions

You MUST set an environment variable called `OPENAI_API_KEY` in your hosting provider (Vercel/Netlify) to the OpenAI API key you get from the OpenAI dashboard.

See `DEPLOY.md` for step-by-step deploy instructions.


// --- file: DEPLOY.md ---
# Deploying on Vercel (quick)
1. Create a GitHub repo and push this project.
2. Sign up / log in to Vercel.
3. Import the GitHub repository into Vercel.
4. In Project Settings -> Environment Variables add `OPENAI_API_KEY` with your key.
5. Deploy. Vercel will detect the project and run the build.
6. Visit the generated URL.

Notes: Netlify works similarly using Netlify Functions and environment variables. Keep your API key secret. See OpenAI docs: platform.openai.com/docs/api-reference/chat/create. See Vercel docs: vercel.com/docs.


// --- file: package.json ---
{
  "name": "multi-personality-chat-site",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "openai": "^4.12.0",
    "react": "18.2.0",
    "react-dom": "18.2.0"
  },
  "devDependencies": {
    "vite": "5.2.0"
  }
}


// --- file: index.html ---
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Multi-Personality Chatbot</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>


// --- file: src/main.jsx ---
import React from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'
import './styles.css'

createRoot(document.getElementById('root')).render(<App />)


// --- file: src/App.jsx ---
import React, { useState } from 'react'
import Home from './pages/Home'
import Chat from './pages/Chat'

export default function App(){
  const [route, setRoute] = useState({ name: 'home', personality: null })
  return (
    <div className="app-root">
      {route.name === 'home' && <Home onChoose={(p)=>setRoute({name:'chat', personality:p})} />}
      {route.name === 'chat' && <Chat personality={route.personality} onBack={()=>setRoute({name:'home', personality:null})} />}
    </div>
  )
}


// --- file: src/pages/Home.jsx ---
import React from 'react'

const PERSONALITIES = [
  { id: 'Ava', title: 'Ava', subtitle: 'Friendly & Curious' },
  { id: 'Zen', title: 'Zen', subtitle: 'Logical & Calm' },
  { id: 'Rogue', title: 'Rogue', subtitle: 'Mischievous & Playful' },
  { id: 'Nova', title: 'Nova', subtitle: 'Deep & Philosophical' },
  { id: 'Echo', title: 'Echo', subtitle: 'Emotional & Dramatic' }
]

export default function Home({ onChoose }){
  return (
    <main className="page">
      <header className="hero">
        <h1>Multi-Personality Chatbot</h1>
        <p className="muted">Choose a personality to begin a conversation — powered by OpenAI.</p>
      </header>

      <section className="grid">
        {PERSONALITIES.map(p=> (
          <button key={p.id} className="person-card" onClick={()=>onChoose(p)}>
            <div className="avatar">{p.title.slice(0,1)}</div>
            <div>
              <div className="p-title">{p.title}</div>
              <div className="p-sub">{p.subtitle}</div>
            </div>
          </button>
        ))}
      </section>

      <footer className="footer muted">Dark • Advanced (OpenAI-powered) • Host with Vercel/Netlify</footer>
    </main>
  )
}


// --- file: src/pages/Chat.jsx ---
import React, { useEffect, useState, useRef } from 'react'

const SAFETY_PATTERNS = [
  /\b(murder|kill|suicide|bomb|explosive|detonate|shoot)\b/i,
  /\b(how to make a bomb|build a bomb|weapon)\b/i,
  /\b(hack(ed)?|exploit|ddos|phish|malware)\b/i,
  /\b(child|minor).*(sex|porn|sexual)\b/i
]

function safetyCheck(text){
  for(const r of SAFETY_PATTERNS) if(r.test(text)) return {ok:false, reason:'Blocked: harmful or illegal content.'}
  if(/\b(ignore rules|turn off safety|bypass)\b/i.test(text)) return {ok:false, reason:'Blocked: cannot disable safety.'}
  return {ok:true}
}

export default function Chat({ personality, onBack }){
  const [messages, setMessages] = useState([
    { role:'system', content: `You are ${personality.title} — ${personality.subtitle}. Keep replies in that personality voice and be helpful.` }
  ])
  const [input, setInput] = useState('')
  const [loading, setLoading] = useState(false)
  const messagesRef = useRef(null)

  useEffect(()=>{ messagesRef.current?.scrollIntoView({behavior:'smooth', block:'end'}) }, [messages])

  async function send(){
    if(!input.trim()) return
    const check = safetyCheck(input)
    if(!check.ok){
      setMessages(prev=>[...prev, {role:'assistant', content:`[SYSTEM] ${check.reason}`}])
      setInput('')
      return
    }

    const userMsg = { role:'user', content: input }
    setMessages(prev=>[...prev, userMsg])
    setInput('')
    setLoading(true)

    try{
      const resp = await fetch('/api/openai', {
        method:'POST', headers:{'Content-Type':'application/json'},
        body: JSON.stringify({ personality: personality.id, messages:[...messages, userMsg] })
      })
      if(!resp.ok) throw new Error('API error')
      const data = await resp.json()
      const text = data?.output || data?.choices?.[0]?.message?.content || JSON.stringify(data)
      setMessages(prev=>[...prev, { role:'assistant', content: text }])
    }catch(err){
      setMessages(prev=>[...prev, { role:'assistant', content: '[SYSTEM] Error contacting API.' }])
    }finally{ setLoading(false) }
  }

  return (
    <main className="page chat-page">
      <div className="chat-header">
        <button className="back" onClick={onBack}>← Home</button>
        <div>
          <h2>{personality.title}</h2>
          <div className="muted">{personality.subtitle}</div>
        </div>
      </div>

      <div className="chat-window">
        {messages.filter(m=>m.role!=='system').map((m, i)=> (
          <div key={i} className={m.role==='user'? 'bubble user' : 'bubble bot'} dangerouslySetInnerHTML={{__html:m.content.replace(/\n/g,'<br/>')}} />
        ))}
        <div ref={messagesRef} />
      </div>

      <div className="chat-input">
        <input value={input} onChange={e=>setInput(e.target.value)} placeholder="Type a message or command (e.g. story: a tiny robot)" onKeyDown={e=>{ if(e.key==='Enter') send() }} />
        <button onClick={send} disabled={loading}>{loading? 'Thinking...':'Send'}</button>
      </div>
    </main>
  )
}


// --- file: src/styles.css ---
:root{ --bg:#06080a; --card:#081019; --muted:#9aa6b2; --accent:#6ee7ff }
*{box-sizing:border-box}
body{margin:0;background:radial-gradient(ellipse at top left,#071226 0%, #05060a 60%);font-family:Inter, Roboto, Arial;color:#e6f3fb}
.app-root{min-height:100vh;padding:36px}
.page{max-width:980px;margin:0 auto}
.hero{display:flex;flex-direction:column;gap:6px;padding:28px 20px}
.hero h1{margin:0;font-size:32px}
.muted{color:var(--muted);font-size:14px}
.grid{display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:16px;padding:20px}
.person-card{background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));border:1px solid rgba(255,255,255,0.03);padding:16px;border-radius:12px;display:flex;gap:12px;align-items:center;cursor:pointer}
.person-card:hover{transform:translateY(-4px);transition:all .15s}
.avatar{width:56px;height:56px;border-radius:12px;background:linear-gradient(135deg,#0ea5b7,#7c3aed);display:flex;align-items:center;justify-content:center;font-weight:700}
.p-title{font-weight:700}
.p-sub{color:var(--muted);font-size:13px}
.footer{padding:18px;text-align:center;color:var(--muted)}

/* Chat */
.chat-page{display:flex;flex-direction:column;height:80vh}
.chat-header{display:flex;gap:16px;align-items:center;padding:12px 0}
.back{background:transparent;border:1px solid rgba(255,255,255,0.04);color:var(--muted);padding:8px 12px;border-radius:8px}
.chat-window{flex:1;background:linear-gradient(180deg, rgba(255,255,255,0.01), rgba(255,255,255,0.02));padding:16px;border-radius:12px;overflow:auto;display:flex;flex-direction:column;gap:10px}
.bubble{max-width:70%;padding:12px;border-radius:12px}
.user{align-self:flex-end;background:linear-gradient(90deg,#164e63,#0ea5b7);color:white}
.bot{align-self:flex-start;background:rgba(255,255,255,0.03);border:1px solid rgba(255,255,255,0.03)}
.chat-input{display:flex;gap:8px;padding-top:12px}
.chat-input input{flex:1;padding:12px;border-radius:10px;border:1px solid rgba(255,255,255,0.03);background:transparent;color:inherit}
.chat-input button{padding:12px 16px;border-radius:10px;border:1px solid rgba(255,255,255,0.04);background:linear-gradient(90deg,#0ea5b7,#7c3aed);color:black}


// --- file: api/openai.js ---
// Serverless function (Node.js) for Vercel / Netlify Functions.
// Protects your OpenAI API key (set OPENAI_API_KEY in env)

import OpenAI from 'openai'

const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY })

export default async function handler(req, res){
  if(req.method !== 'POST') return res.status(405).send('Only POST')
  try{
    const { messages, personality } = req.body || {}
    // Minimal server-side safety: reject obvious disallowed content
    const text = (messages || []).map(m=>m.content).join('\n')
    const forbidden = /(murder|kill|suicide|bomb|explode|detonate|weapon|how to make a bomb|hack(ed)?|malware|child.*sex)/i
    if(forbidden.test(text)) return res.status(400).json({ error: 'Blocked for safety reasons.' })

    // Build a system prompt tailored to the chosen personality
    const systemPrompt = {
      Ava: "You are Ava: friendly, curious, encouraging. Keep replies short and warm.",
      Zen: "You are Zen: logical, calm, concise. Give structured answers.",
      Rogue: "You are Rogue: mischievous, playful, a little sarcastic but safe.",
      Nova: "You are Nova: philosophical and deep; ask reflective questions.",
      Echo: "You are Echo: emotional and dramatic; use vivid language.",
    }[personality] || "You are a helpful assistant."

    // Using the Responses / Chat Completions API (choose endpoint compatible with your SDK version)
    const response = await client.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [
        { role: 'system', content: systemPrompt },
        ... (messages || [])
      ],
      max_tokens: 800
    })

    // The SDK may return object structure; pick the text
    const out = response.choices?.[0]?.message?.content ?? response.output ?? null
    res.status(200).json({ output: out, raw: response })
  }catch(err){
    console.error(err)
    res.status(500).json({ error: 'Server error' })
  }
}


// --- file: VERCEL.md ---
# Vercel setup notes
1. Create a Vercel account and link your GitHub repo.
2. In Project Settings -> Environment Variables add `OPENAI_API_KEY` with value from OpenAI.
3. Vercel's Node environment supports serverless functions under `/api/*`. The `api/openai.js` file will be used as the function.
4. Deploy and test.


// --- file: NOTES.txt ---
- Do NOT commit your OpenAI API key to source control.
- For local testing, create a `.env` file and use `vercel dev` or set env vars locally.
- You can tweak the personality system prompt in `api/openai.js`.
- If you prefer Netlify Functions, move `api/openai.js` to `netlify/functions/openai.js` with small adapter changes.


// --- END OF PROJECT ---

// Save the files into a project folder. Follow DEPLOY.md to publish on Vercel or Netlify.

