# 討論區

一個小小的公開討論區：直接打字留言，大家都能看、能回覆。留言會存成 GitHub issue（label `board`），所以跨瀏覽器、跨裝置都一致，不需要任何伺服器。

<div id="board">
  <p class="board-status">載入中…</p>
</div>

<form id="new-post">
  <h2>新增留言</h2>
  <input id="post-name" type="text" maxlength="40" placeholder="署名（可留空，預設用你的 GitHub ID）" />
  <textarea id="post-body" rows="4" required maxlength="2000" placeholder="要打什麼？"></textarea>
  <button type="submit">發佈（會到 GitHub 建立一條貼文）</button>
  <p class="board-note">發佈需要已登入 GitHub：會開啟 github.com 的新建 issue 表單並自動帶入內容（未登入會先要求登入）。</p>
</form>

## 關於這個討論區

- 留言 = 一條 label 為 `board` 的 GitHub issue；回覆 = 該 issue 的 comments。
- 頁面每 5 分鐘自動重新整理一次；載入失敗多半是 GitHub API 的未登入速率上限（每 IP 每小時 60 次），稍後再試即可。
- 想長期使用、要更順手的討論介面，可以開啟 repo 的 Discussions 並換成 [Giscus](https://giscus.app/)（需在 GitHub 上安裝 giscus app），屆時替換掉本頁的 `<script>` 區塊即可。

<style>
  #board { max-width: 720px; }
  #new-post { max-width: 720px; }
  .board-status { color: #666; font-style: italic; }
  .board-thread { border: 1px solid #ddd; border-radius: 8px; padding: 10px 14px; margin: 12px 0; }
  .board-card { background: #f6f6f6; border-radius: 6px; padding: 8px 12px; margin: 8px 0 8px 16px; }
  .board-meta { font-size: 0.85em; color: #555; }
  .board-author { font-weight: 600; }
  .board-link { font-size: 0.85em; }
  .board-text { white-space: pre-wrap; word-wrap: break-word; margin: 6px 0; }
  .board-note { font-size: 0.8em; color: #888; }
  #new-post textarea { width: 100%; box-sizing: border-box; font: inherit; margin: 8px 0; }
  #new-post input { font: inherit; }
</style>

<script>
(function () {
  var API = 'https://api.github.com';
  var REPO = 'Zhufacheng/zhufacheng.github.io';
  var LABEL = 'board';
  var NEW_URL = 'https://github.com/' + REPO + '/issues/new?labels=' + LABEL;
  var board = document.getElementById('board');

  function el(tag, cls, text) {
    var n = document.createElement(tag);
    if (cls) { n.className = cls; }
    if (text) { n.textContent = text; }
    return n;
  }

  function timeAgo(iso) {
    var s = Math.max(0, (Date.now() - new Date(iso).getTime()) / 1000);
    if (s < 60) { return '剛才'; }
    if (s < 3600) { return Math.floor(s / 60) + ' 分鐘前'; }
    if (s < 86400) { return Math.floor(s / 3600) + ' 小時前'; }
    return Math.floor(s / 86400) + ' 天前';
  }

  function renderComment(c, node) {
    var card = el('div', 'board-card');
    card.appendChild(el('div', 'board-meta', ((c.user && c.user.login) || '?') + ' · ' + timeAgo(c.created_at)));
    card.appendChild(el('p', 'board-text', c.body || ''));
    node.appendChild(card);
  }

  function renderThread(issue, node) {
    var card = el('div', 'board-thread');
    var meta = el('div', 'board-meta');
    var login = issue.user && issue.user.login;
    var a = el('a', 'board-author', login || '?');
    a.href = 'https://github.com/' + login;
    a.target = '_blank';
    a.rel = 'noopener';
    meta.appendChild(a);
    meta.appendChild(el('span', 'board-time', ' · ' + timeAgo(issue.created_at)));
    var link = el('a', 'board-link', ' 開啟討論串 ↗');
    link.href = issue.html_url;
    link.target = '_blank';
    link.rel = 'noopener';
    meta.appendChild(link);
    card.appendChild(meta);
    card.appendChild(el('p', 'board-text', issue.title.replace(/^\[board\]\s*/, '')));
    node.appendChild(card);
    fetch(API + '/repos/' + REPO + '/issues/' + issue.number + '/comments?per_page=30')
      .then(function (r) { if (!r.ok) { throw new Error(String(r.status)); } return r.json(); })
      .then(function (comments) {
        comments.forEach(function (c) { renderComment(c, card); });
      })
      .catch(function () { card.appendChild(el('p', 'board-note', '（載入回覆失敗）')); });
  }

  function load() {
    var status = el('p', 'board-status', '載入中…');
    board.innerHTML = '';
    board.appendChild(status);
    fetch(API + '/repos/' + REPO + '/issues?state=open&labels=' + LABEL + '&per_page=50')
      .then(function (r) {
        if (r.status === 403) { throw new Error('rate-limit'); }
        if (!r.ok) { throw new Error(String(r.status)); }
        return r.json();
      })
      .then(function (issues) {
        board.innerHTML = '';
        if (!issues.length) {
          board.appendChild(el('p', 'board-status', '還沒有留言，第一個留言就是你！'));
          return;
        }
        issues.sort(function (x, y) { return x.created_at < y.created_at ? 1 : -1; });
        issues.forEach(function (issue) { renderThread(issue, board); });
      })
      .catch(function (e) {
        board.innerHTML = '';
        board.appendChild(el('p', 'board-status', '載入失敗（' + e.message + '）。GitHub API 未登入時每 IP 每小時限 60 次，稍後再試。'));
      });
  }

  document.getElementById('new-post').addEventListener('submit', function (ev) {
    ev.preventDefault();
    var name = document.getElementById('post-name').value.trim();
    var body = document.getElementById('post-body').value.trim();
    if (!body) { return; }
    var title = '[board] ' + (name ? name + '：' : '') + body.slice(0, 60);
    var url = NEW_URL + '&title=' + encodeURIComponent(title) + '&body=' + encodeURIComponent(body);
    window.open(url, '_blank');
  });

  load();
  setInterval(load, 300000);
})();
</script>
