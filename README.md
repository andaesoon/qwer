<!doctype html>
<html lang="ko">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Low-cost Voice Translate (Browser STT + Cloud Translate/TTS)</title>
  <style>
    body{font-family:system-ui,-apple-system,Segoe UI,Roboto,"Apple SD Gothic Neo",sans-serif;max-width:900px;margin:24px auto;padding:0 16px}
    h1{font-size:18px;margin:0 0 10px}
    .row{display:flex;gap:10px;flex-wrap:wrap;align-items:center;margin:10px 0}
    select,button,label{font-size:14px}
    select,button{padding:10px 12px;border:1px solid #ddd;border-radius:10px;background:#fff}
    button{cursor:pointer}
    button:disabled{opacity:.5;cursor:not-allowed}
    .box{border:1px solid #e6e6e6;border-radius:14px;padding:12px;min-height:70px;white-space:pre-wrap}
    .label{font-weight:700;margin:14px 0 6px}
    .muted{color:#666;font-size:13px}
    .pill{display:inline-block;padding:4px 10px;border:1px solid #eee;border-radius:999px;background:#fafafa;font-size:12px}
  </style>
</head>
<body>
  <h1>🎙️ 말하면 번역 (비용 최소화)</h1>
  <div class="muted">실시간 자막(interim)은 무료(브라우저 STT). 번역/음성은 “확정 문장(final)”만 호출해서 과금 최소화.</div>

  <div class="row">
    <span class="pill" id="status">Idle</span>

    <label>말하는 언어
      <select id="srcSpeech">
        <option value="th-TH" selected>태국어</option>
        <option value="ko-KR">한국어</option>
        <option value="en-US">영어</option>
        <option value="ja-JP">일본어</option>
        <option value="zh-CN">중국어(간체)</option>
      </select>
    </label>

    <label>번역 대상
      <select id="tgt">
        <option value="ko">한국어</option>
        <option value="th">태국어</option>
        <option value="en" selected>영어</option>
        <option value="ja">일본어</option>
        <option value="zh">중국어</option>
      </select>
    </label>

    <label>TTS
      <select id="tts">
        <option value="none">안 함</option>
        <option value="ko">한국어</option>
        <option value="th" selected>태국어</option>
        <option value="en">영어</option>
        <option value="ja">일본어</option>
        <option value="zh">중국어</option>
      </select>
    </label>
  </div>

  <div class="row">
    <button id="start">시작</button>
    <button id="stop" disabled>중지</button>

    <label class="muted">
      <input type="checkbox" id="autoSpeak" checked />
      확정 문장 자동 읽기
    </label>

    <label class="muted">
      문장 최대 길이(과금 컷)
      <select id="maxLen">
        <option value="120" selected>120자</option>
        <option value="200">200자</option>
        <option value="300">300자</option>
      </select>
    </label>
  </div>

  <div class="label">인식(실시간, 무료)</div>
  <div id="live" class="box"></div>

  <div class="label">번역(확정 문장만 과금)</div>
  <div id="translated" class="box"></div>

  <div class="label">로그</div>
  <div id="log" class="box muted" style="min-height:50px"></div>

  <audio id="player" style="display:none"></audio>

<script>
  // ✅ 네 Cloud Function/Run 프록시 URL로 바꿔
  const API = {
    translateEndpoint: "https://YOUR_CLOUD_FUNCTION_URL/translate",
    ttsEndpoint: "https://YOUR_CLOUD_FUNCTION_URL/tts"
  };

  const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
  if (!SpeechRecognition) alert("이 브라우저는 SpeechRecognition을 지원하지 않습니다. Chrome으로 실행하세요.");

  const $ = (id) => document.getElementById(id);

  const statusEl = $("status");
  const liveEl = $("live");
  const trEl = $("translated");
  const logEl = $("log");
  const player = $("player");

  const srcSpeechEl = $("srcSpeech");
  const tgtEl = $("tgt");
  const ttsEl = $("tts");
  const autoSpeakEl = $("autoSpeak");
  const maxLenEl = $("maxLen");

  const startBtn = $("start");
  const stopBtn = $("stop");

  let rec = null;
  let running = false;

  // 비용 절감: 동일 문장 번역 재호출 방지 캐시
  const cache = new Map(); // key: `${src}|${tgt}|${text}` -> {translatedText, ts}

  function setStatus(s){ statusEl.textContent = s; }
  function log(s){ logEl.textContent = (logEl.textContent ? logEl.textContent + "\n" : "") + s; }

  async function postJSON(url, body){
    const res = await fetch(url, {
      method: "POST",
      headers: {"Content-Type":"application/json"},
      body: JSON.stringify(body),
    });
    if (!res.ok) throw new Error(await res.text().catch(()=>String(res.status)));
    return res.json();
  }

  function b64ToBlob(b64, mime="audio/mpeg"){
    const bytes = atob(b64);
    const arr = new Uint8Array(bytes.length);
    for (let i=0;i<bytes.length;i++) arr[i] = bytes.charCodeAt(i);
    return new Blob([arr], {type:mime});
  }

  async function speakCloud(text, lang){
    if (lang === "none") return;
    if (API.ttsEndpoint.includes("YOUR_CLOUD_FUNCTION_URL")) return;

    // 비용 절감: 너무 긴 건 읽지 않기(원하면 조절)
    const maxLen = Number(maxLenEl.value || 120);
    const clipped = text.length > maxLen ? text.slice(0, maxLen) : text;

    const data = await postJSON(API.ttsEndpoint, { text: clipped, lang });
    if (!data.audioContent) return;

    const blob = b64ToBlob(data.audioContent, data.mimeType || "audio/mpeg");
    const url = URL.createObjectURL(blob);
    player.src = url;
    try { await player.play(); } catch(e) {}
    player.onended = () => URL.revokeObjectURL(url);
  }

  async function translateFinal(text){
    const maxLen = Number(maxLenEl.value || 120);
    const trimmed = (text || "").trim();
    if (!trimmed) return;

    // 비용 절감: 과도하게 길면 잘라서 번역(혹은 아예 skip)
    const clipped = trimmed.length > maxLen ? trimmed.slice(0, maxLen) : trimmed;

    const key = `${srcSpeechEl.value}|${tgtEl.value}|${clipped}`;
    if (cache.has(key)) {
      const cached = cache.get(key);
      trEl.textContent = cached.translatedText;
      if (autoSpeakEl.checked) await speakCloud(cached.translatedText, ttsEl.value);
      return;
    }

    if (API.translateEndpoint.includes("YOUR_CLOUD_FUNCTION_URL")) {
      trEl.textContent = "(translateEndpoint 미설정) " + clipped;
      return;
    }

    setStatus("Translating...");
    const data = await postJSON(API.translateEndpoint, { text: clipped, target: tgtEl.value });
    const translatedText = data.translatedText || "";
    cache.set(key, { translatedText, ts: Date.now() });

    trEl.textContent = translatedText;
    setStatus("Listening...");

    if (autoSpeakEl.checked) await speakCloud(translatedText, ttsEl.value);
  }

  function start(){
    running = true;
    logEl.textContent = "";
    liveEl.textContent = "";
    trEl.textContent = "";

    rec = new SpeechRecognition();
    rec.lang = srcSpeechEl.value;
    rec.interimResults = true;
    rec.continuous = true;

    rec.onresult = async (e) => {
      let interim = "";
      let finalText = "";

      for (let i = e.resultIndex; i < e.results.length; i++) {
        const txt = e.results[i][0].transcript;
        if (e.results[i].isFinal) finalText += txt;
        else interim += txt;
      }

      if (interim) {
        // ✅ 실시간 느낌은 여기서 나옴(하지만 번역 호출 안 해서 0원)
        liveEl.textContent = interim;
        setStatus("Listening...");
      }

      if (finalText) {
        liveEl.textContent = "";
        // ✅ 과금은 final에만
        try {
          await translateFinal(finalText);
        } catch (err) {
          log("translate/tts error: " + (err?.message || err));
          setStatus("Listening...");
        }
      }
    };

    rec.onerror = (err) => {
      log("stt error: " + (err?.error || err));
    };

    rec.onend = () => {
      // 크롬이 가끔 끊어서 “실시간”이 끊기는 걸 막기 위한 자동 재시작
      if (running) {
        try { rec.start(); } catch(e) {}
      }
    };

    try { rec.start(); } catch(e) { log("start error: " + e); }

    startBtn.disabled = true;
    stopBtn.disabled = false;
    setStatus("Listening...");
    log("STT: browser (free) / Translate+TTS: final only (paid)");
  }

  function stop(){
    running = false;
    setStatus("Stopped");
    try { rec && rec.stop(); } catch(e) {}
    startBtn.disabled = false;
    stopBtn.disabled = true;
  }

  startBtn.onclick = start;
  stopBtn.onclick = stop;
</script>
</body>
</html>
