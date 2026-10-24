<script lang="ts">
  import type { PageData } from './$types';

  let { data }: { data: PageData } = $props();
  let filter = $state<'all' | 'good' | 'careful'>('all');

  const filters = [
    { key: 'all', label: 'すべて' },
    { key: 'good', label: '吉のお告げ' },
    { key: 'careful', label: '慎重なお告げ' }
  ] as const;

  const visibleRecords = $derived(data.records.filter((record) => {
    if (filter === 'all') return true;
    const cautious = record.rank === '凶' || record.rank === '大凶';
    return filter === 'careful' ? cautious : !cautious;
  }));

  const formatDate = (iso: string) =>
    new Date(iso).toLocaleString('ja-JP', {
      year: 'numeric',
      month: 'long',
      day: 'numeric',
      hour: '2-digit',
      minute: '2-digit'
    });

  const fortuneTones: Record<string, string> = {
    大吉: '#b18b41', 中吉: '#b54c42', 小吉: '#b56d40', 吉: '#4a7965',
    末吉: '#60768a', 凶: '#7f7184', 大凶: '#5e6477'
  };
</script>

<svelte:head>
  <title>おみくじ帳｜福みくじ</title>
  <meta name="description" content="これまでに引いたおみくじの結果や開運アイテムを見返す、おみくじ帳。" />
</svelte:head>

<div class="history-page">
  <div class="history-hero">
    <div class="hero-pattern" aria-hidden="true">✿</div>
    <a href="/" class="back-link"><span aria-hidden="true">←</span> おみくじ広場へ戻る</a>
    <div class="hero-label">YOUR FORTUNE JOURNAL</div>
    <h1>おみくじ<span>帳</span></h1>
    <p>引いたおみくじは、今日のあなたの小さな記録。<br />これまでのお告げを、そっと振り返ってみましょう。</p>
    <div class="hero-count"><strong>{data.records.length}</strong><span>件の記録を表示中</span></div>
  </div>

  {#if !data.storageReady}
    <div class="banner" role="note">
      <span class="banner-icon" aria-hidden="true">i</span>
      <span>現在は一時保存モードです。記録はサーバーの再起動などで消える場合があります。永続化にはUpstash Redisの接続が必要です。</span>
    </div>
  {/if}

  {#if data.records.length === 0}
    <div class="empty-card">
      <div class="empty-decoration" aria-hidden="true"><span>福</span></div>
      <span class="empty-en">A NEW CHAPTER</span>
      <h2>まだ、まっさらなおみくじ帳。</h2>
      <p>最初の一枚を引いて、<br />あなたの小さな開運記録を始めましょう。</p>
      <a href="/" class="home-button">✦ はじめてのおみくじを引く <span aria-hidden="true">→</span></a>
    </div>
  {:else}
    <div class="list-header">
      <div><span class="list-eyebrow">FORTUNE ARCHIVE</span><h2>お告げの記録</h2></div>
      <div class="filters" role="group" aria-label="おみくじ記録の表示フィルター">
        {#each filters as item (item.key)}
          <button type="button" class:active={filter === item.key} aria-pressed={filter === item.key} onclick={() => (filter = item.key)}>{item.label}</button>
        {/each}
      </div>
    </div>
    {#if visibleRecords.length === 0}
      <div class="filtered-empty">
        <p>この種類のお告げは、まだありません。</p>
        <button type="button" onclick={() => (filter = 'all')}>すべての記録に戻る →</button>
      </div>
    {:else}
      <ol class="record-list">
        {#each visibleRecords as record (record.id)}
          <li class="record" style="--rank-tone: {fortuneTones[record.rank] ?? '#b73535'}">
            <div class="record-stamp" aria-label="運勢 {record.rank}"><span>{record.rank}</span></div>
            <div class="record-content">
              <div class="record-top"><span class="record-id">OMIKUJI · {record.id.slice(0, 6).toUpperCase()}</span><time datetime={record.drawnAt}>{formatDate(record.drawnAt)}</time></div>
              <h3>{record.message}</h3>
              <div class="lucky-row">
                <span><b>◉</b> {record.luckyColor}</span>
                <span><b>✳</b> {record.luckyItem}</span>
                <span><b>♯</b> {record.luckyNumber}</span>
              </div>
            </div>
          </li>
        {/each}
      </ol>
    {/if}
  {/if}
  <div class="return-cta">
    <span aria-hidden="true">✦</span>
    <p>運だめしに、もう一度。</p>
    <a href="/">おみくじを引きに行く <span aria-hidden="true">→</span></a>
  </div>
</div>

<style>
  .history-page { max-width: 1020px; padding: 46px 24px 10px; margin: 0 auto; }
  .history-hero { position: relative; background: radial-gradient(ellipse at 75% 10%,#365262,#182d3a 62%); border-radius: 18px; overflow: hidden; text-align: center; padding: 30px 20px 39px; color: #fff3dc; box-shadow: 0 18px 36px #192e3923; }
  .history-hero::before { content: ''; position: absolute; inset: 12px; border: 1px solid #dfc49142; border-radius: 10px; pointer-events: none; }
  .hero-pattern { position: absolute; right: -10px; bottom: -220px; font: 440px var(--serif); color: #ffffff0a; pointer-events: none; line-height: 1; }
  .back-link { display: block; width: fit-content; margin: 0 auto 25px; color: #e4ccac; text-decoration: none; font-size: 12px; }
  .back-link:hover { text-decoration: underline; }
  .back-link span { margin-right: 8px; }
  .hero-label { position: relative; font: 10px Georgia,serif; letter-spacing: .3em; color: #d4b980; }
  h1 { position: relative; font: 700 45px var(--serif); letter-spacing: .19em; margin: 8px 0 14px; line-height: 1.3; }
  h1 span { color: #d9b676; }
  .history-hero p { position: relative; color: #e3e0d8; font-size: 12px; letter-spacing: .06em; margin: 0; line-height: 2; }
  .hero-count { position: relative; display: flex; justify-content: center; gap: 8px; align-items: baseline; margin-top: 23px; color: #dbcaad; }
  .hero-count strong { color: white; font: 37px Georgia,serif; }
  .hero-count span { font-size: 11px; }
  .banner { display: flex; align-items: start; gap: 12px; background: #fff9ed; border: 1px solid #e9d9bb; border-radius: 10px; color: #927a58; font-size: 11px; line-height: 1.8; padding: 13px 16px; margin-top: 22px; }
  .banner-icon { display: grid; place-items: center; flex: none; border: 1px solid #bb9a62; color: #bb9a62; width: 18px; height: 18px; border-radius: 50%; font: 12px Georgia,serif; margin-top: 2px; }
  .list-header { display: flex; justify-content: space-between; align-items: end; gap: 15px; flex-wrap: wrap; margin: 47px 0 19px; }
  .list-eyebrow { color: #a78b60; font: 10px Georgia,serif; letter-spacing: .25em; }
  .list-header h2 { margin: 4px 0 0; font: 700 26px var(--serif); letter-spacing: .09em; color: #283a41; }
  .filters { display: flex; gap: 7px; flex-wrap: wrap; }
  .filters button { background: #fffdf8; border: 1px solid #e4dbc9; border-radius: 999px; padding: 8px 13px; font-size: 11px; color: #888a81; font-weight: 700; }
  .filters button:hover { border-color: #b28a68; }
  .filters button.active { background: var(--night); color: white; border-color: var(--night); }
  .record-list { list-style: none; padding: 0; margin: 0; display: grid; gap: 13px; }
  .record { display: flex; align-items: stretch; gap: 22px; background: #fffdf8; border: 1px solid #e8dfd1; border-radius: 13px; padding: 20px 24px; box-shadow: 0 7px 24px #15232c07; }
  .record-stamp { display: grid; place-items: center; flex: none; width: 86px; min-height: 99px; border-radius: 5px; border: 2px solid var(--rank-tone); outline: 1px solid var(--rank-tone); outline-offset: -7px; color: var(--rank-tone); background: #faf4e9; }
  .record-stamp span { font: 700 29px var(--serif); writing-mode: vertical-rl; letter-spacing: .1em; }
  .record-content { flex: 1; min-width: 0; align-self: center; }
  .record-top { display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 6px; margin-bottom: 8px; color: #a99e8c; font-size: 10px; }
  .record-id { font: 10px Georgia,serif; letter-spacing: .14em; }
  .record-content h3 { font: 600 17px/1.8 var(--serif); color: #304146; margin: 0 0 13px; }
  .lucky-row { display: flex; flex-wrap: wrap; gap: 7px; }
  .lucky-row span { padding: 4px 11px; background: #f5eee3; border-radius: 999px; color: #8e8474; font-size: 10px; }
  .lucky-row b { font-size: 12px; color: #b39060; margin-right: 4px; }
  .empty-card { margin: 34px 0 0; padding: 58px 24px; border: 1px dashed #d5c3a9; border-radius: 13px; background: #fffbf4; text-align: center; }
  .empty-decoration { display: grid; place-items: center; margin: 0 auto 18px; width: 87px; height: 87px; border-radius: 50%; background: #f0dfc5; border: 8px solid #f7ead8; color: #b03b39; }
  .empty-decoration span { font: 700 43px var(--serif); }
  .empty-en { color: #b09167; font: 10px Georgia,serif; letter-spacing: .22em; }
  .empty-card h2 { font: 700 23px var(--serif); margin: 12px 0; }
  .empty-card p { font-size: 12px; color: #84837d; line-height: 2.1; }
  .home-button { display: inline-flex; gap: 17px; align-items: center; justify-content: center; text-decoration: none; margin-top: 16px; padding: 12px 23px; background: #b73535; color: white; border-radius: 999px; font-size: 12px; font-weight: 700; }
  .home-button:hover { background: #8e3030; }
  .filtered-empty { padding: 50px 14px; text-align: center; background: white; border: 1px solid #e8dfd1; border-radius: 12px; color: #8b8173; }
  .filtered-empty button { background: transparent; border: none; text-decoration: underline; color: #a33c39; }
  .return-cta { text-align: center; margin: 50px auto 20px; color: #9c845b; }
  .return-cta > span { font-size: 22px; }
  .return-cta p { color: #667376; font: 15px var(--serif); margin: 9px 0; }
  .return-cta a { text-decoration: none; color: var(--red); font-size: 12px; font-weight: 700; }
  .return-cta a:hover { text-decoration: underline; }
  @media(max-width:580px) {
    .history-page { padding: 24px 14px 0; }
    .history-hero { padding: 25px 17px 32px; }
    h1 { font-size: 37px; }
    .history-hero p { font-size: 11px; }
    .list-header { margin-top: 32px; }
    .list-header h2 { font-size: 23px; }
    .filters { width: 100%; }
    .filters button { flex: 1; padding: 9px 6px; font-size: 10px; }
    .record { gap: 12px; padding: 14px; }
    .record-stamp { width: 62px; min-height: 82px; }
    .record-stamp span { font-size: 23px; }
    .record-content h3 { font-size: 13px; }
    .record-top { font-size: 9px; }
    .lucky-row span { padding: 3px 7px; font-size: 9px; }
    .empty-card { padding: 45px 18px; }
    .empty-card h2 { font-size: 19px; }
  }
</style>
