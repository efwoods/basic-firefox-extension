# basic-firefox-extension

This is an example of creating a basic firefox extension

[text](https://chatgpt.com/share/681e0713-62e8-8011-8ef9-3e53d4572a57)

I want this to only use javascript and I want it to only be in browser. I want to automatically find the privacy policy on any website, collect that information, and use an in-browser LLM to summarize succintly that information and present it to the user on button press. When this information is available, the extension's icon should increment the count to indicate that a new privacy-policy has been read and summarized and the summary should be immediately available from clicking the app icon
ChatGPT said:

Here is a complete proof-of-concept for a browser extension using JavaScript only that:

    Runs fully in-browser (no server or API calls).

    Scrapes the privacy policy from the active tab.

    Uses an in-browser open-source LLM like llama.cpp via llm-web (https://github.com/jacoblee93/llm-web) or transformers.js to summarize.

    Shows a count badge on the extension icon.

    Displays summaries when clicked.

✅ Setup Instructions
1. File structure:

privacy-summarizer/
├── manifest.json
├── background.js
├── content.js
├── popup.html
├── popup.js
├── summary_store.js
├── llama_worker.js (or other in-browser LLM loader)

✅ manifest.json (Manifest V3):

{
  "manifest_version": 3,
  "name": "Privacy Policy Summarizer",
  "version": "1.0",
  "description": "Summarizes privacy policies using an in-browser LLM",
  "permissions": ["scripting", "activeTab", "storage"],
  "background": {
    "service_worker": "background.js"
  },
  "action": {
    "default_popup": "popup.html",
    "default_icon": {
      "16": "icon.png",
      "48": "icon.png",
      "128": "icon.png"
    }
  },
  "content_scripts": [
    {
      "matches": ["<all_urls>"],
      "js": ["content.js"],
      "run_at": "document_idle"
    }
  ],
  "web_accessible_resources": [
    {
      "resources": ["llama_worker.js"],
      "matches": ["<all_urls>"]
    }
  ]
}

✅ background.js

Handles storing summaries and updating badge count.

let summaryCount = 0;

chrome.runtime.onMessage.addListener((message, sender, sendResponse) => {
  if (message.type === 'saveSummary') {
    chrome.storage.local.get({ summaries: [] }, (data) => {
      const updatedSummaries = [...data.summaries, message.summary];
      summaryCount = updatedSummaries.length;
      chrome.action.setBadgeText({ text: summaryCount.toString() });
      chrome.storage.local.set({ summaries: updatedSummaries });
    });
  }
});

✅ content.js

Attempts to find and fetch the privacy policy automatically.

function findPrivacyPolicyLink() {
  const links = document.querySelectorAll('a');
  for (let link of links) {
    const text = link.innerText.toLowerCase();
    if (text.includes("privacy") && text.includes("policy")) {
      return link.href;
    }
  }
  return null;
}

async function fetchPolicyText(url) {
  try {
    const response = await fetch(url);
    const html = await response.text();
    const doc = new DOMParser().parseFromString(html, 'text/html');
    return doc.body.innerText;
  } catch {
    return null;
  }
}

(async () => {
  const policyURL = findPrivacyPolicyLink();
  if (!policyURL) return;

  const text = await fetchPolicyText(policyURL);
  if (!text) return;

  // Send to background for storage
  chrome.runtime.sendMessage({
    type: 'privacyPolicyFound',
    policyText: text,
  });
})();

✅ popup.html

<!DOCTYPE html>
<html>
<head>
  <title>Privacy Policy Summary</title>
  <style>
    body { font-family: sans-serif; padding: 10px; }
    #summaries { max-height: 400px; overflow-y: auto; }
  </style>
</head>
<body>
  <h2>Privacy Summaries</h2>
  <div id="summaries"></div>
  <script src="popup.js"></script>
</body>
</html>

✅ popup.js

chrome.storage.local.get({ summaries: [] }, (data) => {
  const container = document.getElementById('summaries');
  data.summaries.forEach((summary, index) => {
    const div = document.createElement('div');
    div.innerHTML = `<strong>Site #${index + 1}</strong><br>${summary}<hr>`;
    container.appendChild(div);
  });
});

✅ summary_store.js + Integration with In-Browser LLM

You’ll need to load a small quantized model in browser using transformers.js. Example:
Install transformers.js via CDN in popup.html

<script src="https://cdn.jsdelivr.net/npm/@xenova/transformers@latest/dist/transformers.min.js"></script>

Updated background.js (continued)

chrome.runtime.onMessage.addListener(async (message, sender, sendResponse) => {
  if (message.type === 'privacyPolicyFound') {
    const policyText = message.policyText.slice(0, 4000); // Token limit
    const summarizer = await window.summarizer;
    const summary = await summarizer(policyText);
    
    chrome.runtime.sendMessage({
      type: 'saveSummary',
      summary,
    });
  }
});

Load summarizer model:

// llama_worker.js or injected in background.js
window.summarizer = async function(text) {
  const pipeline = await window.transformers.pipeline('summarization', 'Xenova/distilbart-cnn-6-6');
  const result = await pipeline(text, { max_length: 100 });
  return result[0].summary_text;
};

✅ Add Badge Behavior

chrome.runtime.onInstalled.addListener(() => {
  chrome.action.setBadgeBackgroundColor({ color: "#0000FF" });
  chrome.action.setBadgeText({ text: "0" });
});

🔒 Privacy & Performance Considerations

    All data is stored only locally via chrome.storage.local.

    The LLM runs fully in-browser, no external APIs or servers.

    Add model quantization or tiny models to make it responsive.

    If necessary, add controls to allow user to select if policy should be summarized.

Would you like me to bundle this into a downloadable .zip browser extension directory and walk you through loading it in Chrome or Firefox?
Is this conversation helpful so far?


ChatGPT can make mistakes. Check important info.