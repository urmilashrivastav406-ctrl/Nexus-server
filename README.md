require('dotenv').config();
const express = require('express');
const cors = require('cors');
const { chromium } = require('playwright');
const { createClient } = require('@supabase/supabase-js');

const app = express();
app.use(cors());
app.use(express.json());

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_KEY
);

// Groq API call
async function askGroq(prompt) {
  const res = await fetch('https://api.groq.com/openai/v1/chat/completions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.GROQ_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      model: 'llama3-70b-8192',
      messages: [{ role: 'user', content: prompt }],
      max_tokens: 2000
    })
  });
  const data = await res.json();
  return data.choices[0].message.content;
}

// Health check
app.get('/', (req, res) => {
  res.json({ status: 'NEXUS server running ✅' });
});

// Browse any website
app.post('/browse', async (req, res) => {
  const { url } = req.body;
  try {
    const browser = await chromium.launch({ args: ['--no-sandbox'] });
    const page = await browser.newPage();
    await page.goto(url, { waitUntil: 'domcontentloaded', timeout: 30000 });
    const title = await page.title();
    const text = await page.evaluate(() => document.body.innerText);
    const links = await page.evaluate(() =>
      Array.from(document.querySelectorAll('a'))
        .map(a => ({ text: a.innerText.trim(), href: a.href }))
        .filter(l => l.href.startsWith('http'))
        .slice(0, 20)
    );
    await browser.close();
    res.json({ title, text: text.slice(0, 5000), links });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Click and fill forms
app.post('/interact', async (req, res) => {
  const { url, actions } = req.body;
  try {
    const browser = await chromium.launch({ args: ['--no-sandbox'] });
    const page = await browser.newPage();
    await page.goto(url, { waitUntil: 'domcontentloaded' });
    for (const action of actions) {
      if (action.type === 'click') await page.click(action.selector);
      if (action.type === 'fill') await page.fill(action.selector, action.value);
      if (action.type === 'wait') await page.waitForTimeout(action.ms);
    }
    const result = await page.evaluate(() => document.body.innerText);
    await browser.close();
    res.json({ result: result.slice(0, 5000) });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Save memory
app.post('/memory/save', async (req, res) => {
  const { key, value } = req.body;
  const { error } = await supabase
    .from('memory')
    .upsert({ key, value, updated_at: new Date() });
  res.json({ success: !error });
});

// Load memory
app.get('/memory/:key', async (req, res) => {
  const { data } = await supabase
    .from('memory')
    .select('value')
    .eq('key', req.params.key)
    .single();
  res.json({ value: data?.value || null });
});

// Run parallel tasks
app.post('/parallel', async (req, res) => {
  const { tasks } = req.body;
  try {
    const results = await Promise.all(
      tasks.map(async (task) => {
        const result = await askGroq(task);
        return { task, result };
      })
    );
    res.json({ results });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Ask AI anything
app.post('/ask', async (req, res) => {
  const { prompt } = req.body;
  try {
    const result = await askGroq(prompt);
    res.json({ result });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Scrape specific data
app.post('/scrape', async (req, res) => {
  const { url, selector } = req.body;
  try {
    const browser = await chromium.launch({ args: ['--no-sandbox'] });
    const page = await browser.newPage();
    await page.goto(url, { waitUntil: 'domcontentloaded' });
    const data = await page.evaluate((sel) => {
      return Array.from(document.querySelectorAll(sel))
        .map(el => el.innerText.trim())
        .filter(Boolean);
    }, selector || 'p, h1, h2, h3, li');
    await browser.close();
    res.json({ data: data.slice(0, 100) });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(process.env.PORT || 8080, '0.0.0.0', () => {
  console.log('NEXUS running on port ' + (process.env.PORT || 8080));
});
