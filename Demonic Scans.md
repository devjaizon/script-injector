<!-- Auto Like -->

// Run only on chapter pages
const pattern = /^https:\/\/demonicscans\.org\/title\/[^\/]+\/chapter\/\d+(?:\/\d+)?$/;

if (pattern.test(window.location.href)) {

  // Block window.open only here
  window.open = function() {
    console.log("Blocked an attempt to open a new tab");
    return null;
  };

  // Force all links to open in same tab
  document.addEventListener("click", function(e) {
    const link = e.target.closest("a");
    if (link && link.target === "_blank") {
      e.preventDefault();
      link.target = "_self";
      window.location.href = link.href;
    }
  });

  // Popup notification
  function showPopup(message, success = true) {
    const popup = document.createElement("div");
    popup.textContent = message;
    popup.style.position = "fixed";
    popup.style.bottom = "20px";
    popup.style.right = "20px";
    popup.style.background = success ? "#4caf50" : "#f44336";
    popup.style.color = "white";
    popup.style.padding = "10px 15px";
    popup.style.borderRadius = "8px";
    popup.style.boxShadow = "0 2px 6px rgba(0,0,0,0.3)";
    popup.style.zIndex = "9999";
    popup.style.fontFamily = "sans-serif";
    popup.style.fontSize = "14px";
    document.body.appendChild(popup);
    setTimeout(() => popup.remove(), 3000);
  }

  // Main reaction click logic
  function tryReactions() {
    const container = document.querySelector('.chapter-reactions');
    if (!container) {
      showPopup("❌ Failure: Reactions container not found.", false);
      return;
    }

    const reactions = Array.from(container.querySelectorAll('.reaction'));
    if (reactions.length === 0) {
      showPopup("❌ Failure: No reactions found.", false);
      return;
    }

    // Store initial counts
    const counts = new Map();
    reactions.forEach(r => {
      const span = r.querySelector('.reaction-count');
      const value = span ? parseInt(span.textContent.trim(), 10) : 0;
      counts.set(r, value);
    });

    // Shuffle reactions so it tries in random order
    const shuffled = reactions.sort(() => Math.random() - 0.5);

    function attempt(index) {
      if (index >= shuffled.length) {
        showPopup("❌ Tried all reactions, none increased.", false);
        return;
      }

      const reaction = shuffled[index];
      const span = reaction.querySelector('.reaction-count');
      if (!span) {
        attempt(index + 1);
        return;
      }

      const oldValue = counts.get(reaction);
      reaction.click();

      // Wait a bit to see if it updates
      setTimeout(() => {
        const newValue = parseInt(span.textContent.trim(), 10);
        if (newValue > oldValue) {
          showPopup("✅ Reaction clicked successfully!", true);
        } else {
          attempt(index + 1); // try another
        }
      }, 1000); // 1 second delay to allow DOM update
    }

    attempt(0);
  }

  // Run after small delay to let page load
  setTimeout(tryReactions, 1000);
}


<!-- Auto-buy items -->

// Run only on the exact merchant.php page
if (window.location.href === "https://demonicscans.org/merchant.php") {
  function clickBuyButton() {
    const container = document.querySelector('[data-merch-id="11"]');
    if (container) {
      const button = container.querySelector('button.btn.buy-btn');
      if (button) {
        button.click();
        console.log("✅ Buy button clicked");
      } else {
        console.log("⚠️ Buy button not found inside container");
      }
    } else {
      console.log("⚠️ Container with data-merch-id=6 not found");
    }
  }

  // Run once immediately
  clickBuyButton();

  // Run every 10 seconds
  setInterval(clickBuyButton, 10000);

  // Reload every 2 minutes (120000 ms)
  setInterval(() => {
    console.log("🔄 Reloading page...");
    window.location.reload();
  }, 120000);
}


<!-- Loot monsters -->
(function () {
  'use strict';

  const ACTIVE_PREFIX = 'https://demonicscans.org/active_wave.php?gate=';
  const BATTLE_PREFIX = 'https://demonicscans.org/battle.php?id=';

  const POLL_MS = 1500;
  const RELOAD_DELAY_MS = 4000;
  const MAX_EMPTY_CHECKS = 3;

  let activeWatcher = null;
  let battleWatcher = null;
  let emptyChecks = 0;

  function log(...args) {
    console.log('[AutoLoot]', ...args);
  }

  function wait(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  function urlStartsWith(prefix) {
    return location.href.startsWith(prefix);
  }

  function getActiveUrl() {
    return localStorage.getItem('demonic_active_url') || null;
  }

  function setActiveUrl(url) {
    localStorage.setItem('demonic_active_url', url);
  }

  // NEW: check only inside the first .monster-card
  function findLootJoinButton() {
    const firstCard = document.querySelector('.monster-card');
    if (!firstCard) return null;
    const btn = firstCard.querySelector('.join-btn');
    if (!btn) return null;

    const txt = (btn.innerText || btn.textContent || '').trim();
    if (txt === '💰 Loot' || txt.includes('💰 Loot')) {
      return btn;
    }
    return null;
  }

  function startActiveWatcher() {
    if (activeWatcher) return;
    log('Watching active_wave page...');
    setActiveUrl(location.href); // save exact gate URL
    activeWatcher = setInterval(() => {
      const btn = findLootJoinButton();
      if (btn) {
        log('Found 💰 Loot button in first monster-card, clicking...');
        clearInterval(activeWatcher);
        activeWatcher = null;
        btn.click();
      } else {
        log('No valid Loot button in first monster-card, retrying...');
      }
    }, POLL_MS);
  }

  function startBattleWatcher() {
    if (battleWatcher) return;
    emptyChecks = 0;
    log('Watching battle page...');
    battleWatcher = setInterval(async () => {
      const lootBtn = document.querySelector('#loot-button');
      if (lootBtn) {
        log('Found #loot-button, clicking...');
        lootBtn.click();
        await wait(RELOAD_DELAY_MS);
        location.reload();
      } else {
        emptyChecks++;
        log('No #loot-button, emptyChecks =', emptyChecks);
        if (emptyChecks >= MAX_EMPTY_CHECKS) {
          log('Loot ended. Returning to active_wave page...');
          clearInterval(battleWatcher);
          battleWatcher = null;
          emptyChecks = 0;

          const returnUrl = getActiveUrl();
          if (returnUrl) {
            log('Going back to:', returnUrl);
            location.href = returnUrl;
          } else {
            log('No stored URL found, using fallback prefix');
            location.href = ACTIVE_PREFIX;
          }
        }
      }
    }, POLL_MS);
  }

  function startForCurrentPage() {
    if (urlStartsWith(ACTIVE_PREFIX)) {
      startActiveWatcher();
    } else if (urlStartsWith(BATTLE_PREFIX)) {
      startBattleWatcher();
    } else {
      log('Page not handled:', location.href);
    }
  }

  window.addEventListener('load', startForCurrentPage);
  startForCurrentPage();
})();