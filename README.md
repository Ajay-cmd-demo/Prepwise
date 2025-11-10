/*
Prepwise - Web Starter (single-file React component)

How to use:
- This file is a self-contained React component you can drop into a Next.js project as `pages/index.jsx` or use in a React app.
- It uses Tailwind utility classes (if using CRA, ensure Tailwind is configured or replace classes with your CSS).

Features implemented in this starter:
- Show the project prompt and instructions (the interview test).
- JD paste input to generate keyword list (simple tokenization).
- Text-based interview simulation: user answers to generated questions.
- A lightweight client-side "Prep Report" with heuristics:
  - STAR component checker (looks for S/T/A/R cue phrases)
  - Keyword alignment score with JD
  - Filler word count and pacing hint (by sentence length)
  - Simple sentiment hint using positive/negative word lists
- A single-file, easily-deployable UI for GitHub Pages or Vercel.

Deployment to GitHub Pages (quick):
1. Create a repo `prepwise-web` and add this file as `index.html` or use it inside React.
2. If using this as `index.html` (vanilla), include the built bundle; easier: deploy using Vercel/Netlify by connecting repo.
3. For Next.js: place as `pages/index.jsx`, push to GitHub, connect to Vercel.

Notes:
- This starter is intentionally offline-safe (no external AI calls). Replace the analysis functions with server-side LLM calls later.
- If you want, I can also generate a minimal FastAPI backend that calls OpenAI/Gemini and returns structured scores.
*/

import React, { useState } from 'react';

export default function PrepwiseStarter() {
  const [jdText, setJdText] = useState('');
  const [questions, setQuestions] = useState([
    'Tell me about yourself.',
    'Describe a challenging project you worked on and how you handled it.',
    'Why do you want this role?'
  ]);

  const [answers, setAnswers] = useState({});
  const [report, setReport] = useState(null);

  // simple keyword extraction from JD
  const extractKeywords = (text) => {
    return Array.from(new Set(
      text
        .toLowerCase()
        .replace(/[\W_]+/g, ' ')
        .split(/\s+/)
        .filter(w => w && w.length > 3 && !['this','that','with','your','have','will'].includes(w))
    )).slice(0, 40);
  }

  const generateRoleQuestions = () => {
    const kw = extractKeywords(jdText);
    const extra = kw.slice(0,5).map(k => `How have you used ${k} in your work or projects?`);
    setQuestions([
      'Tell me about yourself.',
      'Walk me through one of your projects and your role in it.',
      ...extra,
      'Describe a time you faced a challenge and what you learned.'
    ]);
  }

  const fillerWords = ['um','uh','like','you know','so','actually','basically','right'];
  const positiveWords = ['improved','success','achieved','increased','reduced','led','built','created'];
  const negativeWords = ['failed','problem','delay','issue','struggled','bug','lost'];

  const checkSTAR = (text) => {
    const lower = text.toLowerCase();
    return {
      situation: /situat|context|background|project/.test(lower),
      task: /task|responsib|role|goal/.test(lower),
      action: /action|I (?:designed|implemented|led|wrote)|implemented|developed|created|built/.test(lower),
      result: /result|impact|outcome|improv|reduc|increas|success|decreased|delivered/.test(lower)
    }
  }

  const scoreAnswer = (ans, jdKeywords) => {
    const text = (ans || '').toLowerCase();
    const words = text.replace(/[\W_]+/g,' ').split(/\s+/).filter(Boolean);

    // keyword overlap
    const kwOverlap = jdKeywords.length === 0 ? 0 : jdKeywords.filter(k => text.includes(k)).length / jdKeywords.length;

    // filler count
    const fillerCount = fillerWords.reduce((c,w)=> c + (text.split(w).length - 1), 0);

    // STAR
    const star = checkSTAR(ans);
    const starScore = (Object.values(star).filter(Boolean).length / 4);

    // sentiment hint (very naive)
    const pos = positiveWords.reduce((c,w)=> c + (text.split(w).length -1),0);
    const neg = negativeWords.reduce((c,w)=> c + (text.split(w).length -1),0);
    const sentiment = pos >= neg ? 'Neutral/Positive' : 'Slightly Negative';

    // readability / pacing cue: avg words per sentence
    const sentences = ans.split(/[.!?]+/).filter(s=>s.trim().length>0);
    const avgWordsPerSentence = sentences.length ? (words.length / sentences.length) : words.length;

    // simple score
    const rawScore = Math.max(0, Math.round((kwOverlap*40 + (1 - Math.min(1, fillerCount/5))*20 + starScore*30 + (avgWordsPerSentence>=8 && avgWordsPerSentence<=25?10:5))*1));

    return {
      kwOverlap: Math.round(kwOverlap*100),
      fillerCount,
      star,
      starScore: Math.round(starScore*100),
      sentiment,
      avgWordsPerSentence: Math.round(avgWordsPerSentence*10)/10,
      score: rawScore
    }
  }

  const runAnalysis = () => {
    const jdKw = extractKeywords(jdText);
    const perQuestion = questions.map((q, idx) => {
      const ans = answers[idx] || '';
      return {
        question: q,
        answer: ans,
        analysis: scoreAnswer(ans, jdKw)
      }
    });

    const overall = {
      totalScore: Math.round(perQuestion.reduce((s,p)=> s + p.analysis.score,0) / Math.max(1, perQuestion.length)),
      keywordHits: jdKw.filter(k => perQuestion.some(p=> p.answer.toLowerCase().includes(k))).length,
      jdKeywordsCount: jdKw.length
    }

    setReport({perQuestion, overall, jdKeywords: jdKw});
  }

  return (
    <div className="min-h-screen bg-gray-50 p-6">
      <div className="max-w-4xl mx-auto bg-white rounded-2xl shadow p-6">
        <h1 className="text-3xl font-bold mb-2">🚀 Prepwise — Live Interview Test (Starter)</h1>
        <p className="text-gray-600 mb-4">Paste a Job Description, generate role-relevant questions, answer them as practice, and get an instant Prep Report (client-side heuristics).</p>

        <section className="mb-6">
          <label className="block text-sm font-medium text-gray-700">Job Description (paste here)</label>
          <textarea
            value={jdText}
            onChange={e=> setJdText(e.target.value)}
            rows={6}
            className="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-2"
            placeholder="Paste JD text here (e.g., requirements, skills, responsibilities)..."
          />
          <div className="mt-2 flex gap-2">
            <button onClick={generateRoleQuestions} className="px-4 py-2 bg-blue-600 text-white rounded">Generate Questions</button>
            <button onClick={() => { setJdText(''); setQuestions([]); }} className="px-4 py-2 bg-gray-200 rounded">Clear</button>
          </div>
        </section>

        <section className="mb-6">
          <h2 className="text-xl font-semibold mb-2">Interview Questions</h2>
          <ol className="list-decimal pl-6 space-y-4">
            {questions.map((q, idx)=> (
              <li key={idx}>
                <div className="mb-1 font-medium">{q}</div>
                <textarea
                  value={answers[idx] || ''}
                  onChange={e=> setAnswers({...answers, [idx]: e.target.value})}
                  rows={3}
                  className="w-full rounded-md border p-2"
                  placeholder="Type your answer here..."
                />
              </li>
            ))}
          </ol>
          <div className="mt-3">
            <button onClick={runAnalysis} className="px-4 py-2 bg-green-600 text-white rounded">Run Prep Report</button>
          </div>
        </section>

        {report ? (
          <section className="mt-6">
            <h2 className="text-2xl font-semibold mb-3">Prep Report</h2>
            <div className="mb-4 p-4 bg-gray-50 rounded">
              <div className="text-lg">Overall Score: <span className="font-bold">{report.overall.totalScore}</span></div>
              <div className="text-sm text-gray-600">JD keywords matched: {report.overall.keywordHits} / {report.overall.jdKeywordsCount}</div>
            </div>

            <div className="space-y-4">
              {report.perQuestion.map((p, i)=> (
                <div key={i} className="p-4 border rounded">
                  <div className="font-semibold">Q: {p.question}</div>
                  <div className="mt-2 text-gray-800">A: {p.answer || <span className="text-gray-400">(no answer)</span>}</div>

                  <div className="mt-3 grid grid-cols-2 gap-4">
                    <div>
                      <div className="text-sm text-gray-600">Score</div>
                      <div className="text-xl font-bold">{p.analysis.score}</div>
                    </div>
                    <div>
                      <div className="text-sm text-gray-600">STAR Coverage</div>
                      <div className="text-sm">S: {p.analysis.star.situation ? '✓' : '—'} • T: {p.analysis.star.task ? '✓' : '—'} • A: {p.analysis.star.action ? '✓' : '—'} • R: {p.analysis.star.result ? '✓' : '—'}</div>
                    </div>
                  </div>

                  <div className="mt-3 text-sm text-gray-700">
                    <div>Keyword Overlap: {p.analysis.kwOverlap}%</div>
                    <div>Filler words: {p.analysis.fillerCount} (try to reduce "um", "like", etc.)</div>
                    <div>Avg words / sentence: {p.analysis.avgWordsPerSentence}</div>
                    <div>Sentiment hint: {p.analysis.sentiment}</div>
                  </div>
                </div>
              ))}
            </div>

            <div className="mt-6 p-4 bg-yellow-50 rounded">
              <h3 className="font-semibold">Quick tips</h3>
              <ul className="list-disc pl-6 text-sm mt-2">
                <li>Use STAR: clearly state Situation → Task → Action → Result.</li>
                <li>Reduce filler words ("um", "like") and keep sentences concise (8–25 words is a good target).</li>
                <li>Reference JD skills explicitly when possible to increase keyword match.</li>
                <li>Prefer outcome-focused language (metrics, impact, delivered results).</li>
              </ul>
            </div>
          </section>
        ) : (
          <div className="mt-6 text-gray-500">Run analysis to see the Prep Report.</div>
        )}

        <footer className="mt-8 text-sm text-gray-500">
          <div>Starter app — offline-safe demo. Replace client-side analysis with server-side LLM calls for production.</div>
        </footer>
      </div>
    </div>
  );
}
# Prepwise
Prepwise
