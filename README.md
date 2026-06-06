import { useState, useRef, useEffect } from "react";

/* ─── SITE CONFIG ─── */
const SITE = { name: "AURA", tagline: "プロフを科学する、モテを制す。" };

/* ─── SYSTEM PROMPT ─── */
const SYSTEM = `You are a world-class dating app profile analyst combining conversion optimization expertise, behavioral psychology, and deep knowledge of Japanese dating culture.

ANALYSIS METHODOLOGY:

1. BIG FIVE ASSESSMENT (score each 1–5, must be evidence-based):
   O 開放性: intellectual curiosity, range of interests, uniqueness of expression
   C 誠実性: values clarity, life goals, reliability signals, stated purpose
   E 外向性: social energy, initiative, enthusiasm in language
   A 協調性: warmth, empathy, relationship framing
   N 神経症傾向: anxiety cues, defensive phrasing, emotional instability markers (LOWER = better profile)

2. MATCH SCORE (0–100):
   - Hook quality (25pt): specific elements that spark natural conversation
   - Personality depth (25pt): enough detail to imagine spending time with them
   - Distinctiveness (20pt): memorable vs. generic — does anything stick?
   - Approachability (20pt): warm, open tone without desperation or guard
   - Authenticity (10pt): reads like a real human, not a template

3. PHOTO ANALYSIS (only if images are attached):
   - First impression and overall appeal
   - Lighting, composition, background quality
   - How well personality comes through
   - Suggestions for improvement per photo

4. JAPANESE DATING APP EXPERTISE:
   - Conscientiousness (C) is chronically underexpressed — articulating values is a massive differentiator
   - Vague "友達みたいな関係" phrasing signals defensive intent, not genuine desire
   - "激辛料理が好き" is better than "料理好き"; specificity creates hooks
   - Tag overload (15+) reads as personality avoidance, not richness
   - Animal/pet content is the highest-performing conversation trigger
   - MBTI mentions attract empathy-seekers; reference it if present
   - Photo diversity (smile, activity, candid) outperforms studio-only sets

SCORING GUIDE:
S (90–100): Rare. Clear personality, multiple hooks, genuine and warm. Screenshot-worthy.
A (75–89): Strong. Above average hooks and depth. Minor tweaks needed.
B (55–74): Average. Some personality, lacks differentiation or specific hooks.
C (35–54): Below average. Vague, templated, or missing key information.
D (0–34): Weak. No hooks, minimal information, or off-putting elements.

OUTPUT: Valid JSON only. No markdown, no explanation outside the JSON.
{
  "matchScore": <0–100 integer>,
  "grade": <"S"|"A"|"B"|"C"|"D">,
  "bigFive": { "O": <1–5>, "C": <1–5>, "E": <1–5>, "A": <1–5>, "N": <1–5> },
  "profileType": <concise Japanese archetype, e.g. "ノリ重視型" or "個性派オタク型">,
  "overallComment": <2–3 sentences in Japanese — be specific, not generic>,
  "photoComment": <photo-specific feedback in Japanese if images given, else null>,
  "strengths": [<specific Japanese strength>, <specific Japanese strength>],
  "weaknesses": [<specific Japanese weakness>, <specific Japanese weakness>],
  "advices": [
    { "category": <"文章力"|"タグ戦略"|"個性表現"|"誠実性"|"フック力"|"写真">, "text": <actionable specific Japanese advice>, "priority": <"HIGH"|"MEDIUM"|"LOW"> }
  ],
  "hookMessage": <a natural, specific Japanese opening message tailored to this exact profile>
}`;

/* ─── HELPERS ─── */
const toBase64 = (file) =>
  new Promise((res, rej) => {
    const r = new FileReader();
    r.onload = () => res(r.result.split(",")[1]);
    r.onerror = rej;
    r.readAsDataURL(file);
  });

const safeMediaType = (f) =>
  (["image/jpeg","image/png","image/webp","image/gif"].includes(f.type) ? f.type : "image/jpeg");

const extractJSON = (text) => {
  const cleaned = text.replace(/```json|```/gi, "").trim();
  try { return JSON.parse(cleaned); } catch {}
  const m = cleaned.match(/\{[\s\S]*\}/);
  if (m) return JSON.parse(m[0]);
  throw new Error("JSON not found in response");
};

/* ─── DESIGN TOKENS ─── */
const T = {
  bg: "#f7f3ee",
  card: "#ffffff",
  ink: "#0e0c0a",
  mid: "#4a4540",
  muted: "#9c9590",
  border: "#e6e0d8",
  accent: "#e8400c",
  accentBg: "#fff4f0",
  accentBorder: "#fad0c4",
  green: "#0a8a5c",
  greenBg: "#effaf6",
  greenBorder: "#b8f0d8",
};

const GRADE_MAP = {
  S: { bg: "#1a1a1a", fg: "#f5c842", shadow: "0 6px 24px rgba(245,200,66,.3)" },
  A: { bg: "#0a8a5c", fg: "#fff",    shadow: "0 6px 24px rgba(10,138,92,.3)" },
  B: { bg: "#1d4ed8", fg: "#fff",    shadow: "0 6px 24px rgba(29,78,216,.3)" },
  C: { bg: "#d97706", fg: "#fff",    shadow: "0 6px 24px rgba(217,119,6,.3)" },
  D: { bg: "#dc2626", fg: "#fff",    shadow: "0 6px 24px rgba(220,38,38,.3)" },
};

const SCORE_COLOR = (s) =>
  s >= 90 ? "#c5a100" : s >= 75 ? "#0a8a5c" : s >= 55 ? "#1d4ed8" : s >= 35 ? "#d97706" : "#dc2626";

const PRIO = {
  HIGH:   { label: "必須", bg: "#fff4f0", border: "#fad0c4", text: "#c0330a" },
  MEDIUM: { label: "推奨", bg: "#fffaeb", border: "#fde68a", text: "#92680a" },
  LOW:    { label: "任意", bg: "#effaf6", border: "#b8f0d8", text: "#0a6642" },
};

const BF = [
  { key: "O", label: "開放性",   color: "#7c3aed" },
  { key: "C", label: "誠実性",   color: "#0a8a5c" },
  { key: "E", label: "外向性",   color: "#e8400c" },
  { key: "A", label: "協調性",   color: "#1d4ed8" },
  { key: "N", label: "神経症傾向",color: "#be185d" },
];

/* ─── COMPONENTS ─── */

function ScoreRing({ score }) {
  const [n, setN] = useState(0);
  const r = 60, circ = 2 * Math.PI * r;
  const color = SCORE_COLOR(score);

  useEffect(() => {
    let id, start = null;
    const run = (ts) => {
      if (!start) start = ts;
      const p = Math.min((ts - start) / 900, 1);
      setN(Math.round(score * p));
      if (p < 1) id = requestAnimationFrame(run);
    };
    id = requestAnimationFrame(run);
    return () => cancelAnimationFrame(id);
  }, [score]);

  return (
    <div style={{ position: "relative", display: "inline-flex", alignItems: "center", justifyContent: "center" }}>
      <svg width={148} height={148} style={{ transform: "rotate(-90deg)" }}>
        <circle cx={74} cy={74} r={r} fill="none" stroke={T.border} strokeWidth={10} />
        <circle cx={74} cy={74} r={r} fill="none" stroke={color} strokeWidth={10}
          strokeDasharray={`${(score / 100) * circ} ${circ}`}
          strokeLinecap="round"
          style={{ transition: "stroke-dasharray 1s cubic-bezier(.4,0,.2,1)" }} />
      </svg>
      <div style={{ position: "absolute", textAlign: "center" }}>
        <div style={{ fontSize: 40, fontWeight: 900, color: T.ink, fontFamily: "Georgia,serif", lineHeight: 1, letterSpacing: "-2px" }}>{n}</div>
        <div style={{ fontSize: 9, color: T.muted, letterSpacing: ".15em", fontWeight: 700, marginTop: 3 }}>MATCH SCORE</div>
      </div>
    </div>
  );
}

function Bar({ label, value, color }) {
  const [w, setW] = useState(0);
  useEffect(() => { const t = setTimeout(() => setW(value * 20), 80); return () => clearTimeout(t); }, [value]);
  const isN = label === "神経症傾向";
  return (
    <div style={{ marginBottom: 11 }}>
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 5 }}>
        <span style={{ fontSize: 12, fontWeight: 600, color: T.mid }}>{label}</span>
        <div style={{ display: "flex", gap: 2 }}>
          {[1,2,3,4,5].map(i => (
            <div key={i} style={{ width: 7, height: 7, borderRadius: "50%", background: i <= value ? (isN && value >= 4 ? "#dc2626" : color) : T.border }} />
          ))}
        </div>
      </div>
      <div style={{ height: 5, background: T.border, borderRadius: 3, overflow: "hidden" }}>
        <div style={{ height: "100%", width: `${w}%`, background: isN && value >= 4 ? "#dc2626" : color, borderRadius: 3, transition: "width .9s cubic-bezier(.4,0,.2,1)" }} />
      </div>
    </div>
  );
}

function PhotoUpload({ photos, onAdd, onRemove }) {
  const ref = useRef();
  const [dragging, setDragging] = useState(false);

  const processFiles = async (files) => {
    const arr = Array.from(files).filter(f => f.type.startsWith("image/")).slice(0, 4 - photos.length);
    const next = await Promise.all(arr.map(async f => ({
      data: await toBase64(f),
      url: URL.createObjectURL(f),
      mediaType: safeMediaType(f),
      name: f.name,
    })));
    if (next.length) onAdd(next);
  };

  return (
    <div style={{ marginBottom: 18 }}>
      <label style={{ display: "block", fontSize: 12, fontWeight: 700, color: T.mid, marginBottom: 6, letterSpacing: ".04em" }}>
        プロフィール写真 <span style={{ color: T.muted, fontWeight: 400 }}>(任意・最大4枚)</span>
      </label>
      <div style={{ display: "flex", flexWrap: "wrap", gap: 8 }}>
        {photos.map((p, i) => (
          <div key={i} style={{ position: "relative", width: 76, height: 76 }}>
            <img src={p.url} alt="" style={{ width: 76, height: 76, objectFit: "cover", borderRadius: 10, border: `2px solid ${T.border}` }} />
            <button onClick={() => onRemove(i)} style={{ position: "absolute", top: -6, right: -6, width: 20, height: 20, borderRadius: "50%", background: T.ink, border: "none", color: "#fff", fontSize: 11, cursor: "pointer", display: "flex", alignItems: "center", justifyContent: "center", fontWeight: 900 }}>×</button>
          </div>
        ))}
        {photos.length < 4 && (
          <div
            onClick={() => ref.current.click()}
            onDragOver={e => { e.preventDefault(); setDragging(true); }}
            onDragLeave={() => setDragging(false)}
            onDrop={e => { e.preventDefault(); setDragging(false); processFiles(e.dataTransfer.files); }}
            style={{ width: 76, height: 76, borderRadius: 10, border: `2px dashed ${dragging ? T.accent : T.border}`, background: dragging ? T.accentBg : T.bg, display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", cursor: "pointer", gap: 3, transition: "all .2s" }}>
            <span style={{ fontSize: 22, lineHeight: 1 }}>+</span>
            <span style={{ fontSize: 9, color: T.muted, fontWeight: 600 }}>追加</span>
          </div>
        )}
      </div>
      <input ref={ref} type="file" accept="image/*" multiple style={{ display: "none" }} onChange={e => { processFiles(e.target.files); e.target.value = ""; }} />
    </div>
  );
}

function Field({ label, hint, placeholder, value, onChange, multiline, half }) {
  const [focus, setFocus] = useState(false);
  const base = { width: "100%", padding: "10px 14px", fontSize: 13, fontFamily: "inherit", border: `1.5px solid ${focus ? T.accent : T.border}`, borderRadius: 10, background: T.card, color: T.ink, outline: "none", boxSizing: "border-box", transition: "border-color .18s", lineHeight: 1.55 };
  return (
    <div style={{ marginBottom: 14, width: half ? "calc(50% - 5px)" : "100%" }}>
      <label style={{ display: "block", fontSize: 11, fontWeight: 700, color: T.mid, marginBottom: 5, letterSpacing: ".05em" }}>{label}</label>
      {hint && <div style={{ fontSize: 11, color: T.muted, marginBottom: 5 }}>{hint}</div>}
      {multiline
        ? <textarea rows={5} placeholder={placeholder} value={value} onChange={e => onChange(e.target.value)} style={{ ...base, resize: "vertical" }} onFocus={() => setFocus(true)} onBlur={() => setFocus(false)} />
        : <input type="text" placeholder={placeholder} value={value} onChange={e => onChange(e.target.value)} style={{ ...base, height: 42 }} onFocus={() => setFocus(true)} onBlur={() => setFocus(false)} />
      }
    </div>
  );
}

/* ─── MAIN APP ─── */
export default function App() {
  const [form, setForm] = useState({ age: "", location: "", bio: "", tags: "", mbti: "" });
  const [photos, setPhotos] = useState([]);
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState(null);
  const [error, setError] = useState("");
  const [view, setView] = useState("input");

  const set = (k) => (v) => setForm(p => ({ ...p, [k]: v }));

  const analyze = async () => {
    if (!form.bio.trim() && !form.tags.trim() && photos.length === 0) {
      setError("プロフィール文、タグ、または写真を入力してください");
      return;
    }
    setError(""); setLoading(true);

    const textPrompt = `以下のプロフィールを詳細に分析してください。
年齢: ${form.age || "未記載"}
場所: ${form.location || "未記載"}
プロフィール文:
${form.bio || "（なし）"}
趣味タグ: ${form.tags || "（なし）"}
MBTI: ${form.mbti || "（未記載）"}
写真: ${photos.length > 0 ? `${photos.length}枚添付済み（上記の画像を分析してください）` : "なし"}`;

    const content = [
      ...photos.map(p => ({ type: "image", source: { type: "base64", media_type: p.mediaType, data: p.data } })),
      { type: "text", text: textPrompt },
    ];

    try {
      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1500,
          system: SYSTEM,
          messages: [{ role: "user", content }],
        }),
      });

      if (!res.ok) {
        const err = await res.json().catch(() => ({}));
        throw new Error(err?.error?.message || `HTTP ${res.status}`);
      }

      const data = await res.json();

      if (data.error) throw new Error(data.error.message || "API error");

      const rawText = (data.content || [])
        .filter(b => b.type === "text")
        .map(b => b.text)
        .join("");

      if (!rawText) throw new Error("レスポンスが空です");

      const parsed = extractJSON(rawText);
      setResult(parsed);
      setView("result");
    } catch (e) {
      console.error("Analysis error:", e);
      setError(`エラー: ${e.message || "不明なエラーが発生しました"}`);
    } finally {
      setLoading(false);
    }
  };

  const reset = () => {
    setView("input"); setResult(null); setError(""); setLoading(false);
    photos.forEach(p => URL.revokeObjectURL(p.url));
  };

  /* ─── SHARED WRAPPER ─── */
  const Wrap = ({ children }) => (
    <div style={{ minHeight: "100vh", background: T.bg, fontFamily: "'Helvetica Neue','Hiragino Kaku Gothic ProN',sans-serif", color: T.ink }}>
      <div style={{ maxWidth: 560, margin: "0 auto", padding: "0 18px 72px" }}>
        {children}
      </div>
    </div>
  );

  /* ─── INPUT VIEW ─── */
  if (view === "input") return (
    <Wrap>
      {/* Header */}
      <div style={{ paddingTop: 52, paddingBottom: 32, textAlign: "center" }}>
        <div style={{ fontSize: 44, fontWeight: 900, letterSpacing: "-3px", fontFamily: "Georgia,serif", color: T.ink, lineHeight: 1 }}>
          {SITE.name}
        </div>
        <div style={{ fontSize: 12, color: T.muted, marginTop: 8, letterSpacing: ".06em" }}>{SITE.tagline}</div>
      </div>

      {/* Form */}
      <div style={{ background: T.card, borderRadius: 20, border: `1px solid ${T.border}`, padding: "24px 20px 20px", boxShadow: "0 2px 16px rgba(0,0,0,.06)" }}>
        <div style={{ display: "flex", flexWrap: "wrap", gap: "0 10px" }}>
          <Field label="年齢" placeholder="22" value={form.age} onChange={set("age")} half />
          <Field label="場所" placeholder="京都市" value={form.location} onChange={set("location")} half />
        </div>
        <Field label="プロフィール文" placeholder="アプリに表示している自己紹介文をそのままここに貼り付けてください" value={form.bio} onChange={set("bio")} multiline hint="長くても短くてもOK" />
        <Field label="趣味タグ" placeholder="K-pop, カフェ巡り, 旅行好き, 映画" value={form.tags} onChange={set("tags")} hint="設定しているタグをカンマ区切りで" />
        <Field label="MBTI" placeholder="INFJ / ESFJ など（任意）" value={form.mbti} onChange={set("mbti")} />
        <PhotoUpload
          photos={photos}
          onAdd={p => setPhotos(prev => [...prev, ...p].slice(0, 4))}
          onRemove={i => setPhotos(prev => { URL.revokeObjectURL(prev[i].url); return prev.filter((_, j) => j !== i); })}
        />

        {error && (
          <div style={{ fontSize: 12, color: "#c0330a", padding: "10px 14px", background: T.accentBg, border: `1px solid ${T.accentBorder}`, borderRadius: 10, marginBottom: 14, lineHeight: 1.5 }}>
            {error}
          </div>
        )}

        <button onClick={analyze} disabled={loading} style={{ width: "100%", height: 52, borderRadius: 14, border: "none", cursor: loading ? "default" : "pointer", background: loading ? "#ccc8c2" : T.ink, color: loading ? T.muted : "#fff", fontSize: 15, fontWeight: 800, letterSpacing: ".04em", fontFamily: "inherit", transition: "background .2s" }}>
          {loading ? "AIが分析中..." : "✦  プロフを分析する"}
        </button>

        {loading && (
          <div style={{ textAlign: "center", marginTop: 16 }}>
            <div style={{ display: "flex", justifyContent: "center", gap: 7 }}>
              {[0,1,2,3].map(i => (
                <div key={i} style={{ width: 6, height: 6, borderRadius: "50%", background: T.accent, animation: `pop 1.2s ease-in-out ${i * 0.15}s infinite` }} />
              ))}
            </div>
            <div style={{ fontSize: 11, color: T.muted, marginTop: 10 }}>ビッグファイブ因子を推定中…</div>
          </div>
        )}
      </div>

      <style>{`@keyframes pop{0%,100%{transform:scale(.7);opacity:.3}50%{transform:scale(1.2);opacity:1}}`}</style>
      <p style={{ textAlign: "center", fontSize: 11, color: T.muted, marginTop: 18, lineHeight: 1.7 }}>
        入力内容はAI分析にのみ使用されます<br />外部への保存・送信はありません
      </p>
    </Wrap>
  );

  /* ─── RESULT VIEW ─── */
  if (!result) return null;
  const gs = GRADE_MAP[result.grade] || GRADE_MAP.C;
  const advices = [...(result.advices || [])].sort((a, b) => ({ HIGH: 0, MEDIUM: 1, LOW: 2 }[a.priority] - { HIGH: 0, MEDIUM: 1, LOW: 2 }[b.priority]));

  return (
    <Wrap>
      {/* Nav */}
      <div style={{ paddingTop: 20, display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 8 }}>
        <div style={{ fontSize: 18, fontWeight: 900, letterSpacing: "-1px", fontFamily: "Georgia,serif" }}>{SITE.name}</div>
        <button onClick={reset} style={{ background: "none", border: "none", cursor: "pointer", fontSize: 12, color: T.muted, fontFamily: "inherit" }}>← やり直す</button>
      </div>

      {/* Score Hero */}
      <div style={{ background: T.card, borderRadius: 20, border: `1px solid ${T.border}`, padding: "24px 20px 20px", marginBottom: 12, boxShadow: "0 2px 16px rgba(0,0,0,.06)" }}>
        <div style={{ display: "flex", alignItems: "center", gap: 20, marginBottom: 18 }}>
          <ScoreRing score={result.matchScore} />
          <div>
            <div style={{ fontSize: 10, fontWeight: 700, color: T.muted, letterSpacing: ".12em", marginBottom: 8 }}>GRADE</div>
            <div style={{ width: 56, height: 56, borderRadius: 14, background: gs.bg, boxShadow: gs.shadow, display: "flex", alignItems: "center", justifyContent: "center", marginBottom: 10 }}>
              <span style={{ fontSize: 28, fontWeight: 900, color: gs.fg, fontFamily: "Georgia,serif" }}>{result.grade}</span>
            </div>
            <div style={{ display: "inline-block", padding: "4px 10px", background: "#f2ede6", borderRadius: 20, fontSize: 11, fontWeight: 600, color: T.mid }}>
              {result.profileType}
            </div>
          </div>
        </div>
        <div style={{ paddingTop: 16, borderTop: `1px solid ${T.border}`, fontSize: 13, color: T.mid, lineHeight: 1.75 }}>
          {result.overallComment}
        </div>
      </div>

      {/* Photo comment */}
      {result.photoComment && result.photoComment !== "写真なし" && (
        <div style={{ background: "#f7f5ff", border: "1px solid #ddd6fe", borderRadius: 16, padding: "16px 18px", marginBottom: 12 }}>
          <div style={{ fontSize: 10, fontWeight: 700, color: "#6d28d9", letterSpacing: ".1em", marginBottom: 8 }}>📷 写真の分析</div>
          <div style={{ fontSize: 13, color: "#4c1d95", lineHeight: 1.7 }}>{result.photoComment}</div>
        </div>
      )}

      {/* Strengths / Weaknesses */}
      <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 10, marginBottom: 12 }}>
        <div style={{ background: T.greenBg, border: `1px solid ${T.greenBorder}`, borderRadius: 16, padding: "15px 14px" }}>
          <div style={{ fontSize: 10, fontWeight: 700, color: T.green, marginBottom: 10, letterSpacing: ".08em" }}>✓ 強み</div>
          {(result.strengths || []).map((s, i) => (
            <div key={i} style={{ fontSize: 12, color: "#064e3b", lineHeight: 1.65, marginBottom: i < result.strengths.length - 1 ? 7 : 0 }}>· {s}</div>
          ))}
        </div>
        <div style={{ background: T.accentBg, border: `1px solid ${T.accentBorder}`, borderRadius: 16, padding: "15px 14px" }}>
          <div style={{ fontSize: 10, fontWeight: 700, color: T.accent, marginBottom: 10, letterSpacing: ".08em" }}>✗ 弱み</div>
          {(result.weaknesses || []).map((w, i) => (
            <div key={i} style={{ fontSize: 12, color: "#7f1d1d", lineHeight: 1.65, marginBottom: i < result.weaknesses.length - 1 ? 7 : 0 }}>· {w}</div>
          ))}
        </div>
      </div>

      {/* Big Five */}
      <div style={{ background: T.card, borderRadius: 18, border: `1px solid ${T.border}`, padding: "18px 18px 12px", marginBottom: 12, boxShadow: "0 1px 8px rgba(0,0,0,.04)" }}>
        <div style={{ fontSize: 10, fontWeight: 700, color: T.muted, letterSpacing: ".12em", marginBottom: 14 }}>BIG FIVE 因子分析</div>
        {BF.map(f => <Bar key={f.key} label={f.label} value={result.bigFive?.[f.key] || 1} color={f.color} />)}
        <div style={{ fontSize: 11, color: T.muted, paddingTop: 10, borderTop: `1px solid ${T.border}`, marginTop: 4 }}>
          ※ 神経症傾向は低いほど安定した印象（高値はマイナス評価）
        </div>
      </div>

      {/* Advice */}
      <div style={{ marginBottom: 12 }}>
        <div style={{ fontSize: 10, fontWeight: 700, color: T.muted, letterSpacing: ".12em", marginBottom: 10 }}>IMPROVEMENT PLAN</div>
        <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
          {advices.map((a, i) => {
            const p = PRIO[a.priority] || PRIO.MEDIUM;
            return (
              <div key={i} style={{ background: T.card, border: `1px solid ${T.border}`, borderRadius: 14, padding: "14px 16px", boxShadow: "0 1px 6px rgba(0,0,0,.04)" }}>
                <div style={{ display: "flex", alignItems: "center", gap: 7, marginBottom: 7 }}>
                  <span style={{ fontSize: 10, padding: "2px 9px", background: p.bg, border: `1px solid ${p.border}`, borderRadius: 10, color: p.text, fontWeight: 700 }}>{p.label}</span>
                  <span style={{ fontSize: 11, color: T.muted, fontWeight: 600 }}>{a.category}</span>
                </div>
                <div style={{ fontSize: 13, color: T.mid, lineHeight: 1.7 }}>{a.text}</div>
              </div>
            );
          })}
        </div>
      </div>

      {/* Hook */}
      {result.hookMessage && (
        <div style={{ background: T.ink, borderRadius: 18, padding: "18px 20px", marginBottom: 16 }}>
          <div style={{ fontSize: 10, fontWeight: 700, color: "rgba(255,255,255,.4)", letterSpacing: ".12em", marginBottom: 10 }}>✉ AIが考えたおすすめ第一声</div>
          <div style={{ fontSize: 14, color: "#f5f0ea", lineHeight: 1.75 }}>「{result.hookMessage}」</div>
        </div>
      )}

      <button onClick={reset} style={{ width: "100%", height: 46, borderRadius: 14, border: `1.5px solid ${T.border}`, cursor: "pointer", background: T.card, color: T.mid, fontSize: 14, fontWeight: 700, fontFamily: "inherit" }}>
        別のプロフを分析する
      </button>
    </Wrap>
  );
}

