<script lang="ts">
  import '../app.css';
  import { page } from '$app/state';

  let { children } = $props();

  const nav = [
    { href: '/', label: 'おみくじを引く', icon: '✦' },
    { href: '/history', label: 'おみくじ帳', icon: '▤' }
  ];
</script>

<svelte:head>
  <meta name="theme-color" content="#172a38" />
</svelte:head>

<div class="app-shell">
  <div class="announcement"><span>✦</span> 一日のはじまりに、ひとさじの福を。 <span>✦</span></div>
  <header class="site-header">
    <div class="header-inner">
      <a class="brand" href="/" aria-label="福みくじ 広場へ戻る">
        <span class="brand-symbol" aria-hidden="true">福</span>
        <span class="brand-copy"><strong>福みくじ</strong><small>FUKU FORTUNE</small></span>
      </a>
      <nav aria-label="メインナビゲーション">
        {#each nav as item (item.href)}
          <a href={item.href} class:active={page.url.pathname === item.href} aria-current={page.url.pathname === item.href ? 'page' : undefined}>
            <span aria-hidden="true">{item.icon}</span> {item.label}
          </a>
        {/each}
      </nav>
    </div>
  </header>

  <main id="main-content">
    {@render children()}
  </main>

  <footer class="site-footer">
    <div class="footer-rule" aria-hidden="true"><span>✿</span></div>
    <div class="footer-inner">
      <span class="footer-brand">福みくじ <i>FUKU FORTUNE</i></span>
      <p>いい日も、そうでない日も。あなたの今日に、小さな福を。</p>
      <small>遊びとしてお楽しみください。占いの結果はランダムです。</small>
    </div>
  </footer>
</div>

<style>
  .app-shell { min-height: 100dvh; display: flex; flex-direction: column; overflow: clip; }
  .announcement {
    min-height: 27px; padding: 5px 16px; text-align: center;
    background: var(--night); color: #e7d3a6; font-size: 10px; letter-spacing: .22em;
  }
  .announcement span { color: #daa957; padding: 0 12px; }
  .site-header { background: rgba(255,253,248,.94); border-bottom: 1px solid #e9e4d9; position: relative; z-index: 5; }
  .header-inner { max-width: 1160px; margin: 0 auto; padding: 18px 26px; display: flex; justify-content: space-between; align-items: center; gap: 24px; }
  .brand { display: inline-flex; gap: 12px; align-items: center; text-decoration: none; }
  .brand-symbol {
    display: grid; place-items: center; width: 45px; height: 45px; border-radius: 14px;
    background: var(--red); border: 2px solid #cf6960; box-shadow: inset 0 0 0 3px #b73535,inset 0 0 0 4px rgba(255,237,210,.7);
    color: #fff6e9; font: 700 24px var(--serif);
  }
  .brand-copy { display: flex; flex-direction: column; line-height: 1.18; gap: 5px; }
  .brand-copy strong { font-family: var(--serif); font-size: 22px; letter-spacing: .11em; font-weight: 800; }
  .brand-copy small { color: #b09a72; font-family: Georgia, serif; letter-spacing: .22em; font-size: 9px; }
  nav { display: flex; align-items: center; gap: 8px; }
  nav a { border-radius: 999px; padding: 10px 17px; text-decoration: none; color: #65737b; font-size: 12px; font-weight: 700; letter-spacing: .03em; transition: background .2s, color .2s, transform .2s; white-space: nowrap; }
  nav a:hover { background: #f2e9dd; transform: translateY(-1px); color: var(--red); }
  nav a.active { color: #fffaf2; background: var(--night); }
  nav a span { color: var(--gold); padding-right: 4px; font-size: 15px; }
  main { width: 100%; flex: 1; }
  .site-footer { margin-top: auto; padding: 36px 24px 40px; text-align: center; color: #7c817e; }
  .footer-rule { max-width: 980px; margin: 0 auto 23px; height: 1px; background: #dfd6c5; position: relative; }
  .footer-rule span { position: absolute; left: 50%; top: 50%; transform: translate(-50%,-50%); padding: 0 22px; background: var(--bg); color: #b9966b; font-size: 18px; }
  .footer-brand { font: 700 21px var(--serif); color: #283b40; letter-spacing: .1em; }
  .footer-brand i { margin-left: 10px; color: #a4a294; font: normal 9px Georgia,serif; letter-spacing: .13em; }
  .footer-inner p { font-size: 12px; letter-spacing: .07em; margin: 10px 0 7px; }
  .footer-inner small { font-size: 10px; color: #a3a6a6; }
  @media(max-width:600px) {
    .announcement { font-size: 9px; letter-spacing: .11em; }
    .announcement span { padding: 0 5px; }
    .header-inner { padding: 13px 16px; gap: 8px; }
    .brand-symbol { width: 38px; height: 38px; border-radius: 12px; font-size: 20px; }
    .brand { gap: 8px; }
    .brand-copy strong { font-size: 17px; letter-spacing: .05em; }
    .brand-copy small { font-size: 7px; letter-spacing: .08em; }
    nav { gap: 3px; }
    nav a { font-size: 10px; padding: 8px 9px; }
    nav a span { display: none; }
    .site-footer { padding-bottom: 24px; }
  }
  @media(max-width:365px) {
    .brand-copy small { display: none; }
    .brand-copy strong { font-size: 16px; }
    nav a { font-size: 9px; padding: 8px; }
  }
</style>
