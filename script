import asyncio
import json
import os
import time
from crawl4ai import AsyncWebCrawler, BrowserConfig, CrawlerRunConfig, CacheMode
from docx import Document

# ----------------- Config -----------------
EMAIL = "ofurich@gmail.com"
PASSWORD = "Peace2478#"

SESSION_ID = "jobhub_session"
LOGIN_URL = "https://www.thejobhub.xyz/auth/login"
AI_ENHANCER_URL = "https://www.thejobhub.xyz/ai-enhancer"

JOB_DESC_FILE = r"C:\Users\OfuRich\Documents\ai jobhub2\Software developer.docx"
CV_FILE = r"C:\Users\OfuRich\Documents\ai jobhub\new cv.docx"
OUTPUT_FILE = "enhanced_cv.md"

# ----------------- Helpers -----------------


def load_docx_text(path):
    if not os.path.exists(path):
        return ""
    if path.lower().endswith(".docx"):
        try:
            doc = Document(path)
            return "\n".join([p.text for p in doc.paragraphs if p.text and p.text.strip()])
        except Exception as e:
            print(f"❌ Error reading .docx ({path}): {e}")
            return ""
    else:
        try:
            with open(path, "r", encoding="utf-8") as f:
                return f.read()
        except Exception as e:
            print(f"❌ Error reading text file ({path}): {e}")
            return ""


# ----------------- JS templates -----------------
LOGIN_JS_TEMPLATE = """
(function(){
    const emailVal = __EMAIL__;
    const passVal  = __PASSWORD__;
    function setReactInput(el, value) {
        if (!el) return;
        try {
            const setter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
            setter.call(el, value);
        } catch(e){ el.value = value; }
        el.dispatchEvent(new Event('input', { bubbles: true }));
        el.dispatchEvent(new Event('change', { bubbles: true }));
    }
    function tryFill() {
        const emailEl = document.querySelector('input[type="email"], input[name*="email" i], input[id*="email" i]');
        const passEl  = document.querySelector('input[type="password"], input[name*="password" i], input[id*="password" i]');
        const btn = Array.from(document.querySelectorAll('button, input[type="submit"]'))
                         .find(b => /(sign|log|submit)/i.test((b.innerText||b.value||"")));
        if (emailEl) setReactInput(emailEl, emailVal);
        if (passEl) setReactInput(passEl, passVal);
        if (emailEl && passEl && (emailEl.value||"").length && (passEl.value||"").length) {
            if (btn) { btn.click(); return true; }
        }
        return false;
    }
    let tries = 0;
    const interval = setInterval(() => {
        tries++;
        if (tryFill() || tries > 20) clearInterval(interval);
    }, 400);
})();
"""

PASTE_JS_TEMPLATE = """
(async function(){
    const JOB_TEXT = __JOB__;
    const CV_TEXT = __CV__;
    function setReactValue(el, value){
        if (!el) return;
        try {
            const setter = Object.getOwnPropertyDescriptor(el.__proto__, 'value').set;
            setter.call(el, value);
        } catch(e){ el.value = value; }
        el.dispatchEvent(new Event('input', { bubbles:true }));
        el.dispatchEvent(new Event('change', { bubbles:true }));
    }
    // Find best candidates for job & cv fields
    const jobEl = document.querySelector("textarea#jobDescription") ||
                  document.querySelector("textarea#job-description") ||
                  document.querySelector("textarea[name='job_description']") ||
                  document.querySelector("textarea");
    const cvEls = Array.from(document.querySelectorAll("textarea")).filter(e=>e !== jobEl);
    const cvEl = document.querySelector("textarea#yourCurrentCvContent") || cvEls[0] || null;

    if (jobEl && JOB_TEXT) { setReactValue(jobEl, JOB_TEXT); }
    if (cvEl && CV_TEXT) { setReactValue(cvEl, CV_TEXT); }

    try { if (jobEl) jobEl.setAttribute("data-locked","true"); } catch(e){}
    try { if (cvEl) cvEl.setAttribute("data-locked","true"); } catch(e){}

    const btn = document.querySelector("button.enhanceMyCv") ||
                Array.from(document.querySelectorAll("button, input[type='submit'], input[type='button']")).find(b=>/enhance/i.test(b.innerText||b.value||""));
    if (btn){ btn.scrollIntoView({behavior:'smooth', block:'center'}); await new Promise(r=>setTimeout(r,200)); btn.click(); return true; }
    return false;
})();
"""

# Parent-page extraction (fallback)
EXTRACT_JS = """
(function(){
    const cssCandidates = [
        "div.__variable_5cfdac__variable_9a8899.antialiased",
        "textarea#_R_e6av5udb_-form-item",
        "div.enhanced-cv-output",
        "div.cv-output",
        "pre",
        "textarea",
        "div[contenteditable='true']"
    ];
    for (const sel of cssCandidates){
        try {
            const el = document.querySelector(sel);
            if (el){
                const txt = (el.value || el.innerText || el.textContent || "").trim();
                if (txt && txt.length>20) return txt;
            }
        } catch(e){}
    }
    return "";
})();
"""

# Iframe extraction
IFRAME_EXTRACT_JS = """
(() => {
    try {
        const iframes = document.querySelectorAll("iframe");
        for (let i = 0; i < iframes.length; i++) {
            let doc;
            try {
                doc = iframes[i].contentDocument || iframes[i].contentWindow.document;
            } catch(e) { continue; }
            if (!doc) continue;

            const candidates = [
                "div.enhanced-cv-output",
                "div.cv-output",
                "pre",
                "textarea",
                "div[contenteditable='true']"
            ];
            for (const sel of candidates) {
                const el = doc.querySelector(sel);
                if (el) {
                    const txt = (el.value || el.innerText || el.textContent || "").trim();
                    if (txt && txt.length > 50) {
                        return txt;
                    }
                }
            }
        }
        return "";
    } catch(e){
        return "";
    }
})();
"""

# ----------------- Main -----------------


async def main():
    browser_cfg = BrowserConfig(headless=False, verbose=True)
    async with AsyncWebCrawler(config=browser_cfg) as crawler:
        print("🔑 Logging in (auto-fill)...")
        login_js = LOGIN_JS_TEMPLATE.replace("__EMAIL__", json.dumps(
            EMAIL)).replace("__PASSWORD__", json.dumps(PASSWORD))
        login_cfg = CrawlerRunConfig(
            session_id=SESSION_ID,
            js_code=login_js,
            wait_for="js:() => location.pathname.includes('/dashboard') || document.body.innerText.includes('Dashboard') || document.body.innerText.includes('Welcome')",
            cache_mode=CacheMode.BYPASS,
        )
        await crawler.arun(url=LOGIN_URL, config=login_cfg)
        print("✅ Login executed.")

        print("🖥 Navigating to AI Enhancer...")
        await crawler.arun(url=AI_ENHANCER_URL, config=CrawlerRunConfig(session_id=SESSION_ID, wait_for="js:() => document.body", cache_mode=CacheMode.BYPASS))
        await asyncio.sleep(1)

        job_desc = load_docx_text(JOB_DESC_FILE)
        cv_text = load_docx_text(CV_FILE)

        if not job_desc or not cv_text:
            print("⚠️ Job description or CV file is missing or empty. Script will still keep the session open; please paste manually and press ENTER to continue.")
            input(
                "👉 After pasting manually into the enhancer page, press ENTER to continue extraction...")

        paste_js = PASTE_JS_TEMPLATE.replace("__JOB__", json.dumps(
            job_desc)).replace("__CV__", json.dumps(cv_text))
        fill_cfg = CrawlerRunConfig(session_id=SESSION_ID, js_code=paste_js,
                                    wait_for="js:() => true", cache_mode=CacheMode.BYPASS)
        print("✍️ Filling job description and CV and clicking Enhance...")
        await crawler.arun(url=AI_ENHANCER_URL, config=fill_cfg)

        print("⏳ Waiting 60 seconds for AI to generate output...")
        await asyncio.sleep(60)

        # First attempt: try extracting from parent page
        print("🔎 Attempting extraction from parent page...")
        extract_cfg = CrawlerRunConfig(
            session_id=SESSION_ID, js_code=EXTRACT_JS, js_only=True, cache_mode=CacheMode.BYPASS)
        res = await crawler.arun(url=AI_ENHANCER_URL, config=extract_cfg)
        enhanced_cv = res.extracted_content.strip() if res and res.extracted_content else ""

        if enhanced_cv and len(enhanced_cv) > 20:
            with open(OUTPUT_FILE, "w", encoding="utf-8") as f:
                f.write(enhanced_cv)
            print(f"💾 Enhanced CV saved -> {OUTPUT_FILE}")
            return

        # If parent extraction failed, try iframe-based JS
        print("⚠️ Parent extraction failed. Trying iframe extraction via JS...")
        iframe_cfg = CrawlerRunConfig(
            session_id=SESSION_ID,
            js_code=IFRAME_EXTRACT_JS,
            js_only=True,
            cache_mode=CacheMode.BYPASS
        )
        res = await crawler.arun(url=AI_ENHANCER_URL, config=iframe_cfg)
        enhanced_cv = res.extracted_content.strip() if res and res.extracted_content else ""

        if enhanced_cv and len(enhanced_cv) > 50:
            with open(OUTPUT_FILE, "w", encoding="utf-8") as f:
                f.write(enhanced_cv)
            print(f"💾 Enhanced CV saved -> {OUTPUT_FILE}")
        else:
            print("❌ Could not extract enhanced CV even from iframe.")

if __name__ == "__main__":
    asyncio.run(main())
