# How Web Notifications Render Across Browsers (Chrome, Firefox, Safari on macOS + Windows)

*Research for Cue Sheet, which calls `new Notification(title, {body, tag})` with no icon and no actions.*
*Date: 2026-07-09.*

**One-line summary:** On desktop, browsers hand your `Notification` off to the native OS notification system (macOS Notification Center, Windows Action/Notification Center), so what your users see is an OS toast stamped with the *browser's* name/logo plus your title and body — icons, images, and action buttons are largely unavailable or ignored for the plain `new Notification()` path on macOS, and several options only work at all from a Service Worker.

---

## TL;DR — Is there a resource that SHOWS how it looks?

There is **no single canonical "gallery" that shows side-by-side screenshots of a web notification in every browser × OS**. The closest and most useful resources, best first:

1. **[web.dev — "Displaying a Notification"](https://web.dev/articles/push-notifications-display-a-notification)** — *The best written reference.* Google's Push team walks through the visual anatomy (title, body, icon, badge, image, actions), shows how the *same* notification renders differently across Chrome/Firefox and across OSes, and states plainly which features the OS drops (e.g. macOS system notifications support neither images nor action buttons). Has inline screenshots. **Start here.**
2. **[push.foo](https://push.foo/)** *(loads; live)* — Interactive playground. Subscribe your own browser, then tweak title/body/icon/badge/image/tag/silent/requireInteraction/renotify and fire a real notification to *see it render on your own machine*. The single best way to see the real thing in your own browser. (Actions were "coming in the next version" as of writing; uses the Push API + service worker.)
3. **[web-push-book.gauntface.com — Notification Examples](https://web-push-book.gauntface.com/demos/notification-examples/)** *(loads; live)* — Button-per-feature demo (icon, badge, image, actions, tag, requireInteraction, silent, renotify, etc.). Buttons grey out when your browser doesn't support a feature, so it doubles as a live capability check. Service-worker based.
4. **[MDN — Using the Notifications API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API/Using_the_Notifications_API)** — Has a screenshot captioned essentially *"To do list — via mdn.github.io — HEY! Your task…"*, which is the clearest single illustration that **the browser stamps the origin/site attribution** into the notification. Also the canonical `tag` replacement example.
5. **[MagicBell Web Push Test](https://www.magicbell.com/test/web-push)** *(was webpushtest.com, now 301-redirects here; loads)* — Send-yourself-a-test tool. Verifies your browser *can* receive push; **does not** show cross-browser visual comparisons.
6. **[Pushpad — "Change the default Chrome icon for notifications"](https://pushpad.xyz/blog/change-the-default-chrome-icon-for-notifications)** — Best source on the no-icon / default-branding question specifically (see §4).

*Dead / weak:* the old JSFiddle/CodePen demos surfaced in search still exist but are just minimal constructor calls, not visual references. No first-party Apple/WebKit or Mozilla page offers a browser-comparison screenshot gallery.

---

## Per-browser / per-OS comparison

| | **Chrome — macOS** | **Chrome — Windows** | **Firefox — macOS** | **Firefox — Windows** | **Safari — macOS** |
|---|---|---|---|---|---|
| **Who draws it** | Native macOS Notification Center | Native Windows toast / Action Center | Native macOS Notification Center (since FF 28) | Native Windows toast (default since FF 111, Mar 2023) | Native macOS Notification Center |
| **App attribution shown** | "Google Chrome" + Chrome logo, always | "Google Chrome" | "Firefox" | "Firefox" | Website name (Safari attributes to the site) |
| **Site/origin shown** | Yes — origin stamped in body area | Yes | Yes | Yes | Yes — "attributed to the website" |
| **Your `icon`** | Shown *alongside* Chrome logo | Shown | Shown | Shown | Website icon (from web app / manifest) |
| **`image` (big picture)** | **Not shown** (macOS system notifs lack it) | Shown (Chrome supports it) | Not shown | Limited | Not shown |
| **`actions` buttons** | **Not shown** (macOS system notifs lack it) | Shown (SW notifs) | Generally not shown | Limited | Not exposed to plain web notifs |
| **`tag` coalescing** | Yes (replaces same-tag) | Yes | Yes | Yes | Yes |
| **`requireInteraction`** | Honored | Honored | **Not honored** (Mozilla declined) | Not honored | Not honored |
| **Auto-dismiss** | ~20 s then dismissed | ~20 s then dismissed | OS-controlled | OS-controlled (moves to Action Center) | OS-controlled (moves to Notification Center) |
| **Position** | Top-right (macOS-controlled) | Bottom-right (Windows-controlled) | Top-right | Bottom-right | Top-right |
| **Plain `new Notification()` works?** | Yes | Yes | Yes | Yes | Yes (constructor works; push needs SW) |

**Key sourcing for the table:**
- Chrome/Firefox delegate to the system notification center, and macOS system notifications support neither images nor action buttons — [web.dev, "Displaying a Notification"](https://web.dev/articles/push-notifications-display-a-notification): *"Chrome and Firefox use the system notifications and notification center on platforms where these are available,"* and on macOS *"system notifications lack support for images and action buttons."* Chrome can opt into custom notifications via `chrome://flags/#enable-system-notifications`.
- Safari relies on macOS Notification Center and attributes to the website — [WebKit, "Meet Web Push"](https://webkit.org/blog/12945/meet-web-push/): *"Safari has always supported local notifications by relying on the macOS Notification Center,"* and per the WebKit 16.1 notes notifications *"look just like any other and are attributed to the website."*
- Firefox on macOS has used Notification Center since Firefox 28 (Mar 2014) and on Windows uses native notifications by default since Firefox 111 (Mar 2023) — [MDN WebExtensions notifications](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/notifications) notes Linux/macOS use system notifications with a XUL fallback; the Windows-111 default is documented widely (e.g. Mozilla release timeline).
- ~20 s desktop auto-dismiss and `requireInteraction` being desktop-only — [Chrome for Developers blog, "requireInteraction"](https://developer.chrome.com/blog/notification-requireInteraction): *"all notifications on desktop will be dismissed after approximately 20 seconds,"* and on Android `requireInteraction` is ignored.
- Firefox declined `requireInteraction` — [Mozilla bug 862395](https://bugzilla.mozilla.org/show_bug.cgi?id=862395) discussion; corroborated in the web.dev ecosystem writeups.

> **Note on precision:** Exact on-screen *position*, the *duration* before a toast slides into the notification center, and precise animations are **OS-controlled and not spec-defined**. The WHATWG spec deliberately leaves presentation to the user agent (see §1). Positions above reflect the platform defaults (macOS top-right, Windows bottom-right) rather than anything a browser guarantees.

---

## 1. Anatomy of a rendered notification

A notification's defined data, per the [WHATWG Notifications spec](https://notifications.spec.whatwg.org/#the-notification-interface): **title, body, direction, language, origin, icon (URL+resource), badge (URL+resource), image (URL+resource), tag, actions (list), silent, renotify, requireInteraction, vibration pattern, timestamp, navigation URL, data, and the service-worker registration.**

What actually reaches the screen and who draws it:

- **Title** — always rendered. The one required argument to the constructor.
- **Body** — rendered below the title.
- **Origin / site attribution** — the spec says *"A notification has an associated origin."* It does **not** mandate visually displaying it, but in practice desktop browsers do: MDN's own example screenshot reads *"To do list — via mdn.github.io — …"* ([Using the Notifications API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API/Using_the_Notifications_API)). Safari *"attributes to the website"* ([WebKit](https://webkit.org/blog/12945/meet-web-push/)).
- **App attribution** — the **OS** stamps which application is displaying the toast (e.g. "Google Chrome", "Firefox"). On macOS this is deliberate and non-removable (§4).
- **Icon** — small image beside title/body when supported. Rendered by the OS toast.
- **Badge** — monochrome glyph; effectively **Chrome-on-Android only** in practice (§5). Not shown on desktop.
- **Image** — large picture; **not** shown by macOS system notifications; shown by Chrome on Windows/Android ([web.dev](https://web.dev/articles/push-notifications-display-a-notification)).
- **Action buttons** — extra buttons; **Service-Worker-only** (§5) and **not** shown on macOS system notifications.

**Who renders what:** the **browser** collects the fields and calls the platform notification API; the **OS** (macOS Notification Center / Windows toast) actually draws the toast, positions it, times it, and stamps the app identity. That is why the same code looks different per OS.

---

## 2. Per-browser + per-OS behavior

See the comparison table above. Highlights:

- **Delegation to native OS:** All three browsers delegate to the platform notification system on desktop where one exists. Chrome/Firefox: [web.dev](https://web.dev/articles/push-notifications-display-a-notification). Safari: [WebKit "Meet Web Push"](https://webkit.org/blog/12945/meet-web-push/). This is why macOS drops `image`/`actions` for these browsers — the limitation is the macOS notification platform, not the browser.
- **Chrome custom-draw escape hatch:** Chrome *can* draw its own richer notifications on all desktop platforms behind `chrome://flags/#enable-system-notifications`, but the default is native ([web.dev](https://web.dev/articles/push-notifications-display-a-notification)).
- **Position:** macOS shows toasts top-right; Windows bottom-right. This is the OS default, not a browser choice, and is **not spec-defined**.
- **Duration:** Chrome desktop auto-dismisses at **~20 s** unless `requireInteraction:true` ([Chrome blog](https://developer.chrome.com/blog/notification-requireInteraction)). For Firefox/Safari the timing is OS-controlled and not documented as a fixed number. After the toast disappears it typically moves into the OS notification center on macOS/Windows (persistent) — though Chrome noted it *removed its own desktop Notification Center* except on ChromeOS, so a dismissed non-persistent Chrome notification may simply be gone ([Chrome blog](https://developer.chrome.com/blog/notification-requireInteraction)).
- **`requireInteraction`:** Honored by Chrome desktop; **not** honored by Firefox (Mozilla explicitly chose not to ship it — [bug 862395](https://bugzilla.mozilla.org/show_bug.cgi?id=862395)); ignored on mobile. MDN flags it as *"not Baseline"* ([MDN requireInteraction](https://developer.mozilla.org/en-US/docs/Web/API/Notification/requireInteraction)).

---

## 3. The `tag` option (Cue Sheet uses this)

`tag` is the **coalescing / replacement** key. Per the [WHATWG spec](https://notifications.spec.whatwg.org/): when a new notification is shown, the UA finds *"the notification in the list of notifications whose tag is not the empty string and is [this] notification's tag, and whose origin is same origin"* and, if found, **replaces the old one** — *"Replace oldNotification with notification, in the list of notifications."* If the platform can't replace in place, the old one is removed.

MDN's plain-language version ([Notification.tag](https://developer.mozilla.org/en-US/docs/Web/API/Notification/tag)): *"more than one notification can share the same tag, linking them together. One notification can then be programmatically replaced with another to avoid the users' screen being filled up with a huge number of similar notifications."*

Behavior in practice ([Using the Notifications API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API/Using_the_Notifications_API)):
- Fire N notifications with the **same tag** ⇒ the user sees effectively **one** (each replaces the prior), instead of a stack of N.
- If the previous same-tag notification is still pending/visible, the new one replaces it; if it was already shown, it closes and the new one shows.
- **Cross-browser:** the same-tag "collapse to one" behavior is consistent across Chrome, Firefox, and Safari (all honor tag). By default replacing a notification does **not** re-alert (no new sound/pop) unless you also set `renotify:true` — and `renotify:true` **requires** a non-empty `tag` or the constructor throws `TypeError` ([MDN Notification() constructor](https://developer.mozilla.org/en-US/docs/Web/API/Notification/Notification)).

**For Cue Sheet:** using a `tag` means repeated fires for the same cue won't pile up — good. If Cue Sheet *wants* each fire to re-alert the user, it must add `renotify:true` (alongside the existing tag).

---

## 4. What a **no-icon** notification looks like (Cue Sheet sets no icon)

Cue Sheet passes no `icon`. Result:

- **The notification still renders** — title, body, and OS/app attribution. No icon field simply means no site-supplied image in that slot.
- **The OS supplies the app's identity regardless.** On **macOS**, the toast shows the **browser's** logo (the Chrome logo for Chrome; Firefox's for Firefox) because *"macOS wants to make it clear what application is displaying the notification"* — and this **cannot be removed or replaced** ([Pushpad](https://pushpad.xyz/blog/change-the-default-chrome-icon-for-notifications)). So a no-icon Cue Sheet notification in Chrome/macOS shows the **Chrome logo**, not Cue Sheet's.
- If you *do* set `icon`, on macOS Chrome shows **both** the Chrome logo *and* your website icon ([Pushpad](https://pushpad.xyz/blog/change-the-default-chrome-icon-for-notifications)).
- **Windows** behavior isn't spelled out by that source, but the pattern is the same: the OS toast is attributed to the browser; a site icon fills the icon slot when provided.
- MDN's `icon` docs do **not** promise a favicon fallback; the [icon property page](https://developer.mozilla.org/en-US/docs/Web/API/Notification/icon) is silent on missing-icon behavior, and the feature is marked *"not Baseline."* So: **don't rely on any automatic favicon substitution** — if no `icon` is set, expect the browser/OS branding to fill that role, and your own branding will be absent.

**Bottom line:** with no icon, users see a generic-looking toast branded as "Google Chrome"/"Firefox"/the site, with your title and body. Adding an `icon` is the single highest-impact change to make it feel like *Cue Sheet*.

---

## 5. Which options are desktop-only vs. Service-Worker-only

From the [MDN Notification() constructor](https://developer.mozilla.org/en-US/docs/Web/API/Notification/Notification) reference and the [WHATWG spec](https://notifications.spec.whatwg.org/):

| Option | Plain `new Notification()` | `ServiceWorkerRegistration.showNotification()` (persistent) | Notes |
|---|---|---|---|
| `title` (arg), `body`, `tag`, `dir`, `lang`, `data`, `silent`, `renotify`, `requireInteraction`, `timestamp`, `icon` | ✅ | ✅ | Constructor accepts all of these |
| **`actions`** | ❌ **Throws `TypeError`** if non-empty | ✅ | *"Only supported for persistent notifications fired using `ServiceWorkerRegistration.showNotification()`"* ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Notification/Notification)) |
| **`image`** | Accepted by constructor, but ⚠️ practically rendered only via the persistent/SW path and dropped by macOS system notifs | ✅ | Not shown on macOS system notifications ([web.dev](https://web.dev/articles/push-notifications-display-a-notification)) |
| **`badge`** | Accepted, but effectively **Chrome-on-Android only** | ✅ | Used *"when there isn't enough space… e.g. the Android Notification Bar"*; monochrome ~96×96; not a desktop feature ([MDN badge](https://developer.mozilla.org/en-US/docs/Web/API/Notification/badge), [web.dev](https://web.dev/articles/push-notifications-display-a-notification)) |

Critical precision points:
- **`actions` genuinely requires a Service Worker.** Calling `new Notification(title, {actions:[…]})` throws `TypeError` — the constructor is defined to reject a non-empty `actions` array ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Notification/Notification)). To get buttons you must register a service worker and call `registration.showNotification(...)`. And even then, **macOS system notifications don't render action buttons** ([web.dev](https://web.dev/articles/push-notifications-display-a-notification)), so on Safari/macOS and Chrome/macOS-default they won't appear.
- **`image` and `badge`** are accepted by the constructor object but have essentially no effect for the plain constructor on desktop macOS; treat them as SW/Android features.
- **Max actions** is *"an implementation-defined integer"* — read `Notification.maxActions` at runtime ([web.dev](https://web.dev/articles/push-notifications-display-a-notification), [WHATWG spec](https://notifications.spec.whatwg.org/)).
- **Mobile:** on nearly all mobile browsers the plain `new Notification()` constructor **throws** — you must use a service worker ([MDN](https://developer.mozilla.org/en-US/docs/Web/API/Notification/Notification)). (Not Cue Sheet's target — it's a desktop app — but worth knowing.)

---

## 6. Existing resources that SHOW the appearance

| Resource | Live? | What it actually shows |
|---|---|---|
| [web.dev — Displaying a Notification](https://web.dev/articles/push-notifications-display-a-notification) | ✅ | Best written+illustrated anatomy; explicit cross-browser/OS differences; states macOS drops image/actions. |
| [push.foo](https://push.foo/) | ✅ | Live playground — fire a real customizable notification in *your* browser (title/body/icon/badge/image/tag/silent/requireInteraction/renotify). Best for seeing the real thing. |
| [web-push-book Notification Examples](https://web-push-book.gauntface.com/demos/notification-examples/) | ✅ | Button-per-feature live demos; greys out unsupported features = live capability check. |
| [MDN — Using the Notifications API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API/Using_the_Notifications_API) | ✅ | Screenshot showing origin attribution ("via mdn.github.io"); canonical tag example. |
| [MagicBell Web Push Test](https://www.magicbell.com/test/web-push) | ✅ (301 from webpushtest.com) | Send-yourself-a-test / capability check. No cross-browser visual gallery. |
| [WebKit — Meet Web Push](https://webkit.org/blog/12945/meet-web-push/) | ✅ | One screenshot of a push notification on macOS; confirms Notification Center + site attribution. |
| [Pushpad — default Chrome icon](https://pushpad.xyz/blog/change-the-default-chrome-icon-for-notifications) | ✅ | Explains + illustrates the Chrome-logo-on-macOS branding and icon/badge on Android. |

**No dedicated cross-browser screenshot gallery exists.** To truly compare, the practical move is: open **push.foo** in each browser you care about and fire the same notification.

---

## Recommendations for Cue Sheet

Cue Sheet currently calls `new Notification(title, {body, tag})` — no icon, no actions. That is a **safe, portable baseline**: it works on Chrome, Firefox, and Safari on desktop without a service worker, and `tag` already prevents notification pile-up.

What users see today, per browser:
- **Chrome/macOS & Firefox/macOS:** a macOS Notification Center toast, top-right, stamped with the **browser's** logo (Chrome/Firefox) — *not* Cue Sheet's — plus your title, body, and the site origin. Auto-dismisses (~20 s in Chrome).
- **Chrome/Windows & Firefox/Windows:** a Windows toast, bottom-right, attributed to the browser; moves into the Action/Notification Center.
- **Safari/macOS:** a Notification Center toast attributed to the website.

Worth adding (in priority order):
1. **`icon`** — the single biggest branding win. With no icon, macOS shows only the Chrome/Firefox logo; adding an icon puts Cue Sheet's mark on the toast (shown *alongside* the browser logo on macOS). Supported on the plain constructor, no service worker needed. Use a reasonably large square PNG (web.dev suggests ~192px to survive high-DPI scaling).
2. **Decide on re-alerting:** you already use `tag` (good — collapses repeats). If a *repeat* of the same cue should re-alert the user (new sound/pop rather than a silent swap), add **`renotify:true`** (requires the tag you already set).
3. **`requireInteraction:true`** *only if* a cue must not be missed — but note it's **Chrome-desktop-only**; Firefox and Safari ignore it, so don't depend on it.

What is **not** worth adding for a plain single-file desktop app:
- **`actions` (buttons):** would force you to add a **service worker** (the constructor throws `TypeError` on non-empty actions), and **macOS still won't render the buttons** anyway. High cost, no payoff on Safari/Chrome-macOS.
- **`image` / `badge`:** effectively no-ops on desktop macOS (image dropped; badge is Chrome-Android-only). Skip them.

**Net:** keep the constructor path; just add an `icon`, and add `renotify:true` if repeated cues should re-alert. Everything else (actions/image/badge/service worker) buys little for a desktop Cue Sheet and costs real complexity.

---

## Sources

- [MDN — Notification() constructor](https://developer.mozilla.org/en-US/docs/Web/API/Notification/Notification) — Authoritative list of constructor options; states `actions` is SW-only and throws `TypeError` on the constructor; `renotify` requires `tag`; mobile constructor throws.
- [MDN — Using the Notifications API](https://developer.mozilla.org/en-US/docs/Web/API/Notifications_API/Using_the_Notifications_API) — Guide with the origin-attribution screenshot and the canonical `tag` replacement example; constructor vs. `showNotification()` difference.
- [MDN — Notification.tag](https://developer.mozilla.org/en-US/docs/Web/API/Notification/tag) — Plain-language definition of tag-based replacement/coalescing.
- [MDN — Notification.icon](https://developer.mozilla.org/en-US/docs/Web/API/Notification/icon) — icon property; notes "not Baseline"; silent on missing-icon fallback.
- [MDN — Notification.badge](https://developer.mozilla.org/en-US/docs/Web/API/Notification/badge) — badge is for constrained displays (Android Notification Bar), monochrome ~96×96; effectively mobile-only.
- [MDN — Notification.requireInteraction](https://developer.mozilla.org/en-US/docs/Web/API/Notification/requireInteraction) — requireInteraction property; flagged "not Baseline."
- [MDN — WebExtensions notifications API](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/API/notifications) — Notes Firefox on Linux/macOS uses system notifications with a XUL fallback.
- [WHATWG Notifications spec](https://notifications.spec.whatwg.org/) — Defines notification data (incl. associated origin), the same-tag "replace oldNotification" rule, max-actions as implementation-defined, and that presentation is up to the user agent (persistent → notification center; non-persistent → not).
- [WHATWG spec — the Notification interface](https://notifications.spec.whatwg.org/#the-notification-interface) — Enumerated notification properties.
- [web.dev — Displaying a Notification](https://web.dev/articles/push-notifications-display-a-notification) — Cross-browser/OS rendering differences; Chrome/Firefox use system notifications; macOS system notifs lack images and action buttons; icon/badge/image sizing; `Notification.maxActions`.
- [Chrome for Developers — Notification requireInteraction](https://developer.chrome.com/blog/notification-requireInteraction) — ~20 s desktop auto-dismiss; requireInteraction is desktop-only (ignored on Android); Chrome removed its own desktop Notification Center (except ChromeOS).
- [WebKit — Meet Web Push](https://webkit.org/blog/12945/meet-web-push/) — Safari uses the macOS Notification Center; push requires a service worker calling the Notifications API; notifications attributed to the website.
- [Mozilla Bugzilla 862395](https://bugzilla.mozilla.org/show_bug.cgi?id=862395) — Firefox discussion on requireInteraction / auto-closing; Mozilla declined to enable it for all sites.
- [Pushpad — Change the default Chrome icon for notifications](https://pushpad.xyz/blog/change-the-default-chrome-icon-for-notifications) — On macOS Chrome shows the Chrome logo (non-removable) plus the website icon; changing branding is only possible on Android via icon+badge.
- [push.foo](https://push.foo/) — Live playground (verified loads) to fire customizable notifications in your own browser.
- [web-push-book — Notification Examples](https://web-push-book.gauntface.com/demos/notification-examples/) — Live per-feature demo (verified loads); greys out unsupported features.
- [MagicBell Web Push Test](https://www.magicbell.com/test/web-push) — Live send-a-test tool (verified; webpushtest.com 301-redirects here); no cross-browser visual gallery.
- [Bennish.net Web Notifications Test](https://www.bennish.net/web-notifications.html) — Old but live minimal test page (Authorize / Show / Show-in-5s buttons); not a rich visual reference.
