<script lang="ts">
  import { enhance } from '$app/forms';
  import { tick } from 'svelte';
  import type { ActionData, PageData } from './$types';

  let { data, form }: { data: PageData; form: ActionData } = $props();
  let drawing = $state(false);
  let wish = $state('今日の運勢');
  let shareMessage = $state('');
  let drawError = $state('');
  let ceremonyPhase = $state(0);
  let tied = $state(false);
  const burstPieces = Array.from({ length: 18 }, (_, i) => i);

  const ceremonyLines = [
    '',
    '願いを、そっと込めて…',
    '福を、呼び寄せています…',
    'お告げの準備ができました',
    'さあ、開運の瞬間です'
  ];

  const wait = (ms: number) => new Promise<void>((resolve) => setTimeout(resolve, ms));
  const prefersLessMotion = () => window.matchMedia('(prefers-reduced-motion: reduce)').matches;
  const moveTo = (id: string) => {
    const el = document.getElementById(id);
    if (el) el.scrollIntoView({ behavior: prefersLessMotion() ? 'auto' : 'smooth', block: 'start' });
  };


  const wishes = [
    { name: '今日の運勢', symbol: '✦' },
    { name: '恋愛・ご縁', symbol: '♡' },
    { name: '仕事・学び', symbol: '✎' },
    { name: '新しい挑戦', symbol: '↗' }
  ];

  const rankTone: Record<string, string> = {
    大吉: '#b18b41',
    中吉: '#bd5845',
    小吉: '#b56d40',
    吉: '#4a7965',
    末吉: '#60768a',
    凶: '#7f7184',
    大凶: '#5e6477'
  };
  const rankWords: Record<string, string> = {
    大吉: '最高の追い風が吹いています',
    中吉: 'うれしい予感に満ちた一日',
    小吉: '小さな幸せを見つける日',
    吉: '穏やかな幸運が訪れそう',
    末吉: '未来の福を育てるとき',
    凶: '丁寧に進めばきっと大丈夫',
    大凶: 'ここから運気は上向きに'
  };

  async function shareFortune() {
    if (!form?.result) return;
    const result = form.result;
    const text = `【福みくじ】${wish}：${result.rank}\n${result.message}\nラッキーカラー：${result.lucky.color} ／ アイテム：${result.lucky.item} ／ 数字：${result.lucky.number}\n${location.origin}`;
    try {
      if (navigator.share) {
        await navigator.share({ title: '福みくじのお告げ', text });
        shareMessage = 'お告げを共有しました';
      } else if (navigator.clipboard?.writeText) {
        await navigator.clipboard.writeText(text);
        shareMessage = 'お告げをコピーしました！';
      } else {
        shareMessage = 'お使いのブラウザでは共有ができません';
      }
    } catch (error) {
      if ((error as Error).name !== 'AbortError') shareMessage = '共有できませんでした';
    }
  }
</script>

<svelte:head>
  <title>福みくじ｜今日のあなたに、小さな福を</title>
  <meta name="description" content="願いを込めておみくじを引こう。アニメーションとともに、七つの運勢と今日のお告げが届く無料のミニ占い。" />
</svelte:head>

<div class="home">
  <section class="hero" aria-labelledby="hero-title">
    <div class="hero-copy">
      <div class="eyebrow"><span class="eyebrow-line"></span> A LITTLE BIT OF LUCK <span class="eyebrow-line"></span></div>
      <div class="seal"><span>開</span><span>運</span></div>
      <h1 id="hero-title">今日という日に、<br /><em>小さな福</em>を。</h1>
      <p class="hero-lead">少し立ち止まって、運だめし。<br />箱をひと振りすると、今のあなたに贈る<br class="desktop-break" />ちいさなお告げが届きます。</p>
      <div class="blessing"><span aria-hidden="true">✿</span> 願いごとをひとつ選びましょう</div>
      <div class="wish-options" role="group" aria-label="願いごとのテーマを選ぶ">
        {#each wishes as option (option.name)}
          <button type="button" class:chosen={wish === option.name} aria-pressed={wish === option.name} onclick={() => { wish = option.name; shareMessage = ''; }}>
            <span aria-hidden="true">{option.symbol}</span> {option.name}
          </button>
        {/each}
      </div>
      <p class="wish-note">※ テーマは演出用です。運勢はランダムに抽選されます。</p>
      <form method="POST" action="?/draw" use:enhance={() => {
        drawing = true;
        tied = false;
        shareMessage = '';
        drawError = '';
        ceremonyPhase = 1;
        const startedAt = Date.now();
        // The draw itself remains server-side. Only presentation is delayed for the ceremony.
        const firstStep = window.setTimeout(() => { if (drawing) ceremonyPhase = 2; }, 550);
        const secondStep = window.setTimeout(() => { if (drawing) ceremonyPhase = 3; }, 1300);
        window.requestAnimationFrame(() => moveTo('ritual-stage'));

        return async ({ result, update }) => {
          try {
            const minimumCeremony = prefersLessMotion() ? 0 : 2100;
            await wait(Math.max(0, minimumCeremony - (Date.now() - startedAt)));
            window.clearTimeout(firstStep);
            window.clearTimeout(secondStep);
            if (result.type === 'success') {
              ceremonyPhase = 4;
              await update({ reset: false });
              drawing = false;
              await tick();
              // Two frames let the new fortune card enter the layout before scrolling.
              window.requestAnimationFrame(() => {
                window.requestAnimationFrame(() => moveTo('fortune-result'));
              });
            } else {
              await update({ reset: false });
              drawError = 'おみくじを引けませんでした。通信状態を確認して、もう一度お試しください。';
            }
          } catch (error) {
            console.error('[fuku-mikuji] Draw presentation failed', error);
            drawError = 'お告げの表示で問題が発生しました。画面を更新して、もう一度お試しください。';
          } finally {
            window.clearTimeout(firstStep);
            window.clearTimeout(secondStep);
            drawing = false;
            ceremonyPhase = 0;
          }
        };
      }}>
        <button type="submit" class="draw-button" disabled={drawing}>
          <span class="draw-spark" aria-hidden="true">✧</span>
          <span>{drawing ? 'お告げを準備しています…' : form?.result ? 'もう一度、おみくじを引く' : '願いを込めて、おみくじを引く'}</span>
          <span class="draw-arrow" aria-hidden="true">→</span>
        </button>
      </form>
      {#if drawError}<p class="draw-error" role="alert">{drawError}</p>{/if}
      <div class="foot-note"><span class="star-mark" aria-hidden="true">✦</span> おみくじは何度でも引けます <span class="foot-divider">/</span> 全7種類の運勢</div>
    </div>

    <div id="ritual-stage" class="stage" class:drawing class:awakening={ceremonyPhase >= 3} aria-label="おみくじ箱のイラスト">
      <div class="stage-grid" aria-hidden="true"></div>
      <div class="stage-header"><span>福</span><span>招</span><span>き</span><span>の</span><span>社</span></div>
      <div class="moon" aria-hidden="true"></div>
      <span class="petal petal-1" aria-hidden="true">✦</span>
      <span class="petal petal-2" aria-hidden="true">✧</span>
      <span class="petal petal-3" aria-hidden="true">✦</span>
      <span class="petal petal-4" aria-hidden="true">✧</span>
      <div class="torii" aria-hidden="true">
        <div class="torii-roof"></div><div class="torii-top"></div>
        <div class="torii-bar"></div><div class="torii-post left"></div><div class="torii-post right"></div>
        <div class="torii-plaque">開運</div>
      </div>
      <div class="ground-circle" aria-hidden="true"></div>
      <div class="omikuji-art" aria-hidden="true">
        <div class="fortune-stick one"></div>
        <div class="fortune-stick two"></div>
        <div class="fortune-stick three"></div>
        <div class="art-box">
          <span class="art-box-mark">福</span>
          <span class="art-box-caption">おみくじ</span>
        </div>
      </div>
      {#if drawing}
        <div class="ritual-lights" aria-hidden="true">
          {#each burstPieces.slice(0, 12) as piece (piece)}
            <i style="--light-number: {piece}"></i>
          {/each}
        </div>
        <div class="ritual-caption" role="status" aria-live="polite">
          <div class="ritual-line"><span aria-hidden="true">✧</span> {ceremonyLines[ceremonyPhase] || ceremonyLines[2]} <span aria-hidden="true">✧</span></div>
          <div class="ritual-dots" aria-hidden="true"><span class:active={ceremonyPhase >= 1}></span><span class:active={ceremonyPhase >= 2}></span><span class:active={ceremonyPhase >= 3}></span></div>
        </div>
      {/if}
      <div class="stage-footer"><span></span> SHAKE & FIND YOUR FORTUNE <span></span></div>
    </div>
  </section>

  {#if form?.result}
    <section id="fortune-result" class="reveal" aria-labelledby="result-title">
      <div class="reveal-heading"><span class="heading-line"></span><span>✧ お告げが届きました ✧</span><span class="heading-line"></span></div>
      {#key form.record?.id ?? form.result.rank}
      <div class="result-card" class:celebration={form.result.rank === '大吉' || form.result.rank === '中吉'} style="--fortune-tone: {rankTone[form.result.rank] ?? '#b73535'}">
        {#if form.result.rank === '大吉'}
          <div class="blessing-burst" aria-hidden="true">
            {#each burstPieces as piece (piece)}<i style="--index: {piece}"></i>{/each}
          </div>
        {/if}
        <div class="result-upper">
          <div class="paper-corner top-left">✿</div><div class="paper-corner top-right">✿</div>
          <span class="fortune-kicker">FUKU FORTUNE · 御神籤</span>
          <span class="fortune-scroll-mark" aria-hidden="true">✦　開運招福　✦</span>
          <span class="result-topic">{wish}</span>
          <h2 id="result-title">{form.result.rank}</h2>
          <p class="fortune-phrase">{rankWords[form.result.rank] ?? '今日もよい一日を'}</p>
          <div class="ornament-divider"><span>◆</span></div>
          <p class="oracle">{form.result.message}</p>
          {#if form.result.rank === '大吉'}<p class="special-message">✦　今日は、とびきりの福日和！　✦</p>{/if}
        </div>
        <div class="lucky-section">
          <p class="lucky-title">TODAY'S LUCKY CHARMS <span>今日の開運アイテム</span></p>
          <div class="lucky-grid">
            <div class="lucky-item"><span class="lucky-icon" aria-hidden="true">◉</span><small>ラッキーカラー</small><strong>{form.result.lucky.color}</strong></div>
            <div class="lucky-item"><span class="lucky-icon" aria-hidden="true">✳</span><small>ラッキーアイテム</small><strong>{form.result.lucky.item}</strong></div>
            <div class="lucky-item"><span class="lucky-icon" aria-hidden="true">♯</span><small>ラッキーナンバー</small><strong>{form.result.lucky.number}</strong></div>
          </div>
        </div>
        <div class="result-bottom">
          <button class="share-button" type="button" onclick={shareFortune}><span aria-hidden="true">↗</span> お告げをシェア</button>
          <a class="history-link" href="/history">これまでのおみくじを見る <span aria-hidden="true">→</span></a>
          <button class="again-button" type="button" onclick={() => moveTo('hero-title')}>もう一度引く ↑</button>
        </div>
        {#if shareMessage}<p class="share-status" role="status">{shareMessage}</p>{/if}
        {#if form.result.rank === '凶' || form.result.rank === '大凶'}
          <div class="tie-section">
            <p>気になるお告げは、ここに結んで心を切り替えよう。</p>
            <button type="button" class="tie-button" aria-pressed={tied} onclick={() => (tied = !tied)}>{tied ? '✿　気持ちを結びました' : '✿　おみくじを結ぶ'}</button>
            {#if tied}<p class="tie-done" role="status">心機一転。明日はまた、新しい一日です。</p>{/if}
          </div>
        {/if}
        {#if !form.persisted}
          <p class="storage-note">おみくじ帳は現在一時保存モードです。時間が経つと記録が消える場合があります。</p>
        {/if}
      </div>
      {/key}
    </section>
  {:else}
    <section class="guide" aria-label="おみくじの楽しみ方">
      <div class="guide-heading"><span>HOW TO ENJOY</span><h2>福みくじの楽しみ方</h2></div>
      <div class="guide-grid">
        <div><span class="guide-number">01</span><div class="guide-icon" aria-hidden="true">♡</div><h3>願いを選ぶ</h3><p>心に浮かんだテーマを<br />ひとつ選んでみましょう。</p></div>
        <div><span class="guide-number">02</span><div class="guide-icon" aria-hidden="true">✧</div><h3>福を呼び込む</h3><p>おみくじ箱に願いを込めて、<br />今日の運勢を引きます。</p></div>
        <div><span class="guide-number">03</span><div class="guide-icon" aria-hidden="true">✉</div><h3>お告げを受け取る</h3><p>七つの運勢とラッキーな<br />ヒントをお届けします。</p></div>
      </div>
    </section>
  {/if}
  <p class="bottom-invite">今日も、あなたらしい一日になりますように。 <span aria-hidden="true">✿</span></p>
</div>

<style>
  .home { max-width: 1180px; margin: 0 auto; padding: 54px 25px 15px; }
  .hero { display: grid; grid-template-columns: minmax(0,1fr) minmax(410px,.92fr); align-items: center; gap: clamp(30px,6vw,90px); }
  .hero-copy { padding: 16px 0 30px 10px; position: relative; }
  .eyebrow { display: flex; align-items: center; gap: 12px; color: #aa8455; font: 700 10px Georgia,serif; letter-spacing: .25em; white-space: nowrap; }
  .eyebrow-line { width: 24px; height: 1px; background: #c4a97f; }
  .seal { display: flex; position: absolute; right: 4px; top: 83px; flex-direction: column; gap: 1px; color: #c9786a; border: 1px solid #dba99a; border-radius: 3px; padding: 7px 6px; font: 16px var(--serif); line-height: 1; transform: rotate(10deg); opacity: .85; }
  h1 { margin: 30px 0 21px; color: #1a323b; font: 700 clamp(36px,4vw,57px)/1.54 var(--serif); letter-spacing: .055em; white-space: nowrap; }
  h1 em { color: var(--red); font-style: normal; position: relative; }
  h1 em::after { content: ''; height: 8px; position: absolute; left: 0; right: 0; bottom: 3px; background: #e6bfa6; opacity: .46; z-index: -1; transform: rotate(-2deg); }
  .hero-lead { color: #6f7677; font-size: 14px; line-height: 2.4; letter-spacing: .06em; margin: 0 0 30px; }
  .blessing { font-weight: 700; font-size: 13px; color: #34464c; margin-bottom: 15px; }
  .blessing span { color: #c36458; font-size: 19px; margin-right: 9px; }
  .wish-options { display: flex; gap: 9px; flex-wrap: wrap; max-width: 500px; }
  .wish-options button { border: 1px solid #e3d8c7; background: #fffdf8; border-radius: 999px; color: #777b78; padding: 9px 13px; line-height: 1.2; font-size: 11px; font-weight: 700; transition: .22s ease; }
  .wish-options button span { margin-right: 4px; color: #b28d64; font-size: 13px; }
  .wish-options button:hover { border-color: #ba6b5d; transform: translateY(-2px); }
  .wish-options button.chosen { background: #f9e9e4; border-color: #d9a29a; color: #a63534; box-shadow: 0 3px 10px #b735350a; }
  .wish-note { color: #9b9c96; font-size: 10px; margin: 12px 0 25px; }
  .draw-button { width: min(100%,435px); display: flex; justify-content: center; align-items: center; gap: 14px; padding: 19px 17px; color: #fff8e9; border: 1px solid #b9433d; background: linear-gradient(120deg,#aa2e32,#c54943 58%,#a62b2e); border-radius: 13px; font-weight: 800; font-size: 14px; letter-spacing: .08em; box-shadow: 0 7px 0 #832628,0 14px 20px #8e413127; transition: transform .2s, box-shadow .2s, opacity .2s; }
  .draw-button:hover:not(:disabled) { transform: translateY(-3px); box-shadow: 0 10px 0 #832628,0 16px 23px #8e41312c; }
  .draw-button:active:not(:disabled) { transform: translateY(5px); box-shadow: 0 2px 0 #832628; }
  .draw-button:disabled { opacity: .8; cursor: progress; }
  .draw-spark { color: #ffe0a6; font-size: 21px; }
  .draw-arrow { color: #ffddaa; font-size: 22px; }
  .foot-note { margin-top: 27px; color: #92958d; font-size: 11px; letter-spacing: .035em; }
  .foot-note .star-mark { color: var(--gold); font-size: 17px; vertical-align: -2px; margin-right: 4px; }
  .foot-divider { color: #cfbea2; padding: 0 10px; }
  .stage { min-height: 605px; border-radius: 190px 190px 22px 22px; background: radial-gradient(circle at 50% 36%,#395563 0%,#213b48 49%,#162b36 90%); position: relative; isolation: isolate; overflow: hidden; border: 7px solid #fff9ec; box-shadow: 0 20px 48px #2133452b, inset 0 0 0 1px #48606a; }
  .stage::after { content: ''; position: absolute; inset: 11px; border: 1px solid #ffffff1d; border-radius: 170px 170px 14px 14px; pointer-events: none; }
  .stage-grid { position: absolute; inset: 0; opacity: .12; background-image: radial-gradient(circle,#ebd3a9 1px,transparent 1px); background-size: 23px 23px; mask-image: linear-gradient(to bottom,transparent,#000 30%,transparent 100%); }
  .stage-header { position: absolute; top: 32px; left: 0; right: 0; text-align: center; display: flex; justify-content: center; gap: 9px; color: #e6d4b3; font: 500 13px var(--serif); letter-spacing: .14em; z-index: 2; }
  .stage-header span:nth-child(3) { color: #e5aa6f; }
  .moon { width: 278px; height: 278px; position: absolute; border-radius: 50%; background: radial-gradient(circle at 35% 35%, #f5dfb3,#d6b77e 68%,#a2875b); opacity: .84; filter: drop-shadow(0 0 35px #dfc68a55); left: 50%; top: 109px; transform: translateX(-50%); box-shadow: 0 0 0 20px #d8bf8c0b,0 0 0 48px #d8bf8c0a; }
  .torii { position: absolute; width: 325px; height: 260px; left: 50%; top: 187px; transform: translateX(-50%); z-index: 1; filter: drop-shadow(0 16px 7px #101d2b28); }
  .torii-roof { position: absolute; height: 19px; left: 0; right: 0; top: 12px; background: #af3937; border-radius: 45% 45% 5px 5px; transform: perspective(110px) rotateX(11deg); border-top: 4px solid #dd6b57; }
  .torii-top { position: absolute; height: 15px; left: 17px; right: 17px; top: 42px; background: #ac3732; border-bottom: 5px solid #8e292b; }
  .torii-bar { position: absolute; top: 100px; left: 41px; right: 41px; height: 15px; background: #bc4940; }
  .torii-post { position: absolute; width: 23px; top: 48px; height: 210px; background: linear-gradient(90deg,#83282b,#b63d35,#df6c57,#8d2f2d); }
  .torii-post.left { left: 68px; transform: rotate(1deg); }
  .torii-post.right { right: 68px; transform: rotate(-1deg); }
  .torii-plaque { position: absolute; left: 50%; transform: translateX(-50%); top: 52px; background: #192d37; border: 3px solid #bb925b; padding: 7px 8px 5px; color: #e8cc93; font: 700 13px var(--serif); writing-mode: vertical-rl; letter-spacing: .1em; }
  .ground-circle { position: absolute; width: 470px; height: 170px; border-radius: 50%; background: radial-gradient(ellipse,#c18d5e52,transparent 70%); bottom: 30px; left: 50%; transform: translateX(-50%); }
  .omikuji-art { width: 190px; height: 220px; position: absolute; bottom: 99px; left: 50%; transform: translateX(-50%); z-index: 3; transform-origin: 50% 90%; filter: drop-shadow(0 20px 12px #050b1770); }
  .art-box { position: absolute; bottom: 0; left: 25px; width: 145px; height: 154px; background: linear-gradient(100deg,#bb4136,#ab2f2d 50%,#852e30); border: 3px solid #de8060; border-radius: 13px 13px 22px 22px; box-shadow: inset 0 0 0 5px #9e3832,inset 0 0 0 6px #e8b37b; display: flex; flex-direction: column; justify-content: center; align-items: center; }
  .art-box::before { content: ''; position: absolute; height: 17px; left: -8px; right: -8px; top: -13px; background: linear-gradient(90deg,#a33732,#df7458,#ac3831); border: 2px solid #dfa16f; border-radius: 5px; }
  .art-box-mark { font: 700 75px/1.15 var(--serif); color: #f5deb2; text-shadow: 1px 2px 0 #6f2428; }
  .art-box-caption { margin-top: 7px; color: #f0cd9e; font: 12px var(--serif); letter-spacing: .25em; }
  .fortune-stick { position: absolute; width: 13px; height: 132px; bottom: 123px; border-radius: 3px 3px 0 0; background: linear-gradient(90deg,#d1a972,#fff0bd,#cf9f68); border: 1px solid #d2a66b; transform-origin: bottom; z-index: -1; }
  .fortune-stick.one { left: 68px; transform: rotate(-12deg); }
  .fortune-stick.two { left: 92px; transform: rotate(4deg); height: 146px; }
  .fortune-stick.three { left: 119px; transform: rotate(20deg); height: 124px; }
  .petal { position: absolute; color: #efc985; text-shadow: 0 0 18px #ffd58b; font-size: 25px; animation: float 5s ease-in-out infinite; }
  .petal-1 { top: 23%; left: 14%; animation-delay: -1s; }
  .petal-2 { top: 37%; right: 11%; animation-delay: -3s; font-size: 33px; }
  .petal-3 { bottom: 25%; left: 10%; animation-delay: -2s; font-size: 16px; }
  .petal-4 { bottom: 18%; right: 12%; animation-delay: -4s; font-size: 20px; }
  .stage-footer { position: absolute; bottom: 27px; width: 100%; display: flex; align-items: center; justify-content: center; gap: 12px; color: #e5d1ad9c; font: 10px Georgia,serif; letter-spacing: .17em; white-space: nowrap; }
  .stage-footer span { height: 1px; width: 26px; background: #d9b87a88; }
  .stage.drawing .omikuji-art { animation: shake .55s ease-in-out infinite; }
  .stage.drawing .petal { animation-duration: .6s; }
  .reveal { padding: 72px 0 20px; }
  .reveal-heading { display: flex; align-items: center; justify-content: center; gap: 24px; color: #986f51; font: 700 14px var(--serif); letter-spacing: .15em; margin-bottom: 26px; }
  .heading-line { width: 70px; height: 1px; background: #d2bd9b; }
  .result-card { margin: 0 auto; max-width: 800px; background: #fffcf5; border: 1px solid #e4d9c6; border-radius: 12px; box-shadow: 0 20px 50px #342f2410; overflow: hidden; animation: enter .75s cubic-bezier(.2,.7,.2,1) both; }
  .result-upper { position: relative; text-align: center; padding: 42px 32px 34px; background: radial-gradient(circle at 50% 35%,#fffdf9,#faf2e4); }
  .result-upper::before { content: ''; position: absolute; inset: 11px; border: 1px solid #e8d8bc; pointer-events: none; }
  .paper-corner { position: absolute; z-index: 1; top: 17px; color: #cbb18d; }
  .paper-corner.top-left { left: 20px; }
  .paper-corner.top-right { right: 20px; }
  .fortune-kicker { display: block; font: 11px Georgia,serif; letter-spacing: .24em; color: #a28a64; }
  .result-topic { margin: 20px auto 2px; display: block; width: fit-content; background: #f1e5d4; border-radius: 999px; padding: 4px 17px; font-size: 11px; color: #8f7255; }
  .result-upper h2 { margin: 10px 0 0; font: 800 clamp(80px,15vw,125px)/1.35 var(--serif); letter-spacing: .08em; color: var(--fortune-tone); text-shadow: 2px 4px 0 #e9ddcc; }
  .fortune-phrase { font: 700 16px var(--serif); letter-spacing: .08em; color: var(--fortune-tone); margin: 0 0 14px; }
  .ornament-divider { width: 240px; max-width: 70%; height: 1px; margin: 26px auto 24px; background: #d9c8ad; position: relative; }
  .ornament-divider span { position: absolute; top: 50%; left: 50%; transform: translate(-50%,-50%); padding: 0 12px; background: #fcf6e9; color: #ba9567; font-size: 12px; }
  .oracle { color: #3e4a49; font: 500 clamp(15px,2vw,18px)/2.1 var(--serif); letter-spacing: .05em; margin: 0 auto; max-width: 540px; }
  .lucky-section { background: #f6f0e5; border-top: 1px solid #ebdfc9; padding: 25px 24px 32px; }
  .lucky-title { margin: 0 0 18px; text-align: center; font: 11px Georgia,serif; color: #a28251; letter-spacing: .17em; }
  .lucky-title span { display: block; margin-top: 4px; font: 700 11px var(--serif); letter-spacing: .08em; }
  .lucky-grid { display: grid; grid-template-columns: repeat(3,1fr); gap: 13px; }
  .lucky-item { background: #fffcf7; border: 1px solid #e6dccb; border-radius: 10px; padding: 18px 8px; text-align: center; display: flex; flex-direction: column; gap: 4px; }
  .lucky-icon { font-size: 22px; color: #ba8f62; line-height: 1; margin-bottom: 7px; }
  .lucky-item small { font-size: 10px; color: #959187; }
  .lucky-item strong { font: 700 20px var(--serif); color: #243d41; }
  .result-bottom { display: flex; padding: 25px; gap: 16px; justify-content: center; align-items: center; flex-wrap: wrap; }
  .share-button { background: var(--night); border: 1px solid var(--night); color: white; border-radius: 999px; padding: 11px 23px; font-size: 12px; font-weight: 700; }
  .share-button span { margin-right: 7px; color: #e8c386; }
  .share-button:hover { background: #2c4958; }
  .history-link { text-decoration: none; font-size: 12px; color: #946947; padding: 10px; }
  .history-link:hover { text-decoration: underline; }
  .history-link span { margin-left: 5px; }
  .share-status { text-align: center; font-size: 12px; margin: -12px 10px 17px; color: #3f7c69; }
  .storage-note { font-size: 10px; text-align: center; color: #9c968b; padding: 0 18px 20px; margin: 0; }
  .guide { margin: 80px auto 0; max-width: 1000px; }
  .guide-heading { text-align: center; }
  .guide-heading span { font: 10px Georgia,serif; letter-spacing: .24em; color: #b58d58; }
  .guide-heading h2 { font: 700 23px var(--serif); color: #273d44; margin: 7px 0 28px; letter-spacing: .12em; }
  .guide-grid { display: grid; grid-template-columns: repeat(3,1fr); gap: 16px; }
  .guide-grid > div { position: relative; padding: 27px 17px; text-align: center; border: 1px solid #e9dfd0; border-radius: 12px; background: #fffcf6; }
  .guide-number { position: absolute; top: 12px; left: 18px; color: #c8ac83; font: 12px Georgia,serif; }
  .guide-icon { display: grid; place-items: center; width: 55px; height: 55px; border-radius: 50%; background: #f4eae0; color: #bc564c; margin: 0 auto 14px; font: 29px var(--serif); }
  .guide-grid h3 { font: 700 16px var(--serif); letter-spacing: .12em; margin: 0 0 9px; }
  .guide-grid p { font-size: 12px; line-height: 2; color: #7e827d; margin: 0; }
  .bottom-invite { margin: 64px 0 5px; text-align: center; font: 14px var(--serif); letter-spacing: .08em; color: #8a8174; }
  .bottom-invite span { color: #c96c61; margin-left: 6px; }
  @keyframes float { 0%,100% { transform: translateY(0) rotate(0); opacity: .8; } 50% { transform: translateY(-20px) rotate(22deg); opacity: 1; } }
  @keyframes shake { 0%,100% { transform: translateX(-50%) rotate(-8deg); } 50% { transform: translateX(-50%) rotate(8deg); } }
  @keyframes enter { from { opacity: 0; transform: translateY(25px) scale(.98); } to { opacity: 1; transform: translateY(0) scale(1); } }
  @media(max-width:950px) {
    .hero { grid-template-columns: minmax(0,1fr) minmax(320px,.88fr); gap: 26px; }
    .hero-copy { padding-left: 0; }
    h1 { font-size: clamp(30px,4vw,42px); }
    .stage { min-height: 510px; }
    .stage .torii { transform: translateX(-50%) scale(.82); transform-origin: top center; }
    .moon { width: 240px; height: 240px; }
    .omikuji-art { bottom: 87px; }
  }
  @media(max-width:710px) {
    .home { padding: 29px 15px 15px; }
    .hero { display: flex; flex-direction: column; align-items: stretch; gap: 27px; }
    .hero-copy { padding: 8px 3px 0; }
    .eyebrow { font-size: 9px; }
    .seal { right: 16px; top: 82px; }
    h1 { font-size: clamp(35px,8vw,48px); margin: 17px 0 15px; }
    .hero-lead { font-size: 12px; line-height: 2.25; margin-bottom: 23px; }
    .desktop-break { display: none; }
    .wish-options { max-width: none; }
    .wish-options button { padding: 9px 11px; }
    .draw-button { width: 100%; }
    .stage { width: 100%; min-height: 510px; border-radius: 200px 200px 18px 18px; }
    .moon { top: 105px; }
    .reveal { padding-top: 47px; }
    .guide { margin-top: 54px; }
    .guide-grid { gap: 9px; }
    .guide-grid > div { padding: 24px 9px; }
    .guide-grid p { font-size: 10px; }
  }
  @media(max-width:480px) {
    .stage { min-height: 468px; }
    .stage-header { top: 28px; }
    .moon { width: 228px; height: 228px; top: 104px; }
    .torii { top: 185px; }
    .omikuji-art { bottom: 64px; }
    .stage-footer { bottom: 18px; font-size: 8px; }
    .reveal-heading { gap: 9px; font-size: 12px; }
    .heading-line { width: 27px; }
    .result-upper { padding: 38px 20px 30px; }
    .result-upper h2 { font-size: 78px; }
    .fortune-phrase { font-size: 13px; }
    .oracle { font-size: 15px; }
    .lucky-section { padding: 20px 12px 24px; }
    .lucky-grid { gap: 7px; }
    .lucky-item { padding: 12px 3px; }
    .lucky-item small { font-size: 9px; }
    .lucky-item strong { font-size: 16px; }
    .result-bottom { padding: 20px 12px; }
    .guide-grid { grid-template-columns: 1fr; gap: 9px; }
    .guide-grid > div { display: grid; grid-template-columns: 60px 1fr; align-items: center; column-gap: 13px; text-align: left; padding: 16px 22px; }
    .guide-number { top: 12px; right: 15px; left: auto; }
    .guide-icon { grid-row: span 2; margin: 0; }
    .guide-grid h3 { align-self: end; margin-bottom: 2px; }
    .guide-grid p { align-self: start; }
    .guide-grid p br { display: none; }
    .bottom-invite { font-size: 12px; margin-top: 48px; }
  }

  /* Ceremony: movement is decorative; the draw remains server-authoritative. */
  #ritual-stage, #fortune-result { scroll-margin-top: 24px; }
  .draw-error { color: #a52f33; font-size: 12px; margin-top: 16px; line-height: 1.7; }
  .stage.drawing { box-shadow: 0 0 0 6px #e8bf7a50, 0 0 55px #bd985e44, inset 0 0 0 1px #48606a; }
  .stage.drawing .omikuji-art { animation: ceremonial-shake .49s ease-in-out infinite; }
  .stage.drawing .fortune-stick.one { animation: stick-left .9s ease-in-out infinite alternate; }
  .stage.drawing .fortune-stick.two { animation: stick-center .75s ease-in-out infinite alternate; }
  .stage.drawing .fortune-stick.three { animation: stick-right 1.05s ease-in-out infinite alternate; }
  .stage.awakening .moon { animation: moon-awaken 1.15s ease-in-out infinite alternate; }
  .ritual-lights { position: absolute; inset: 0; z-index: 4; pointer-events: none; }
  .ritual-lights i { position: absolute; left: 50%; top: 52%; width: 5px; height: 5px; background: #ffe0a0; border-radius: 50%; box-shadow: 0 0 15px 4px #f6d192d1; --light-angle: calc(var(--light-number) * 30deg); animation: ritual-light 1.6s ease-out infinite; animation-delay: calc(var(--light-number) * -0.14s); }
  .ritual-caption { position: absolute; z-index: 5; left: 30px; right: 30px; bottom: 66px; text-align: center; padding: 14px 12px; background: #162e3de3; border: 1px solid #f5dba177; border-radius: 8px; color: #ffe3aa; backdrop-filter: blur(8px); box-shadow: 0 10px 25px #0004; }
  .ritual-line { font: 700 clamp(12px, 2vw, 16px) var(--serif); letter-spacing: .09em; }
  .ritual-line span { padding: 0 6px; }
  .ritual-dots { display: flex; justify-content: center; gap: 7px; margin-top: 10px; }
  .ritual-dots span { background: #ffffff40; width: 18px; height: 3px; border-radius: 99px; transition: background .2s, width .2s; }
  .ritual-dots span.active { background: #ffd791; width: 30px; }
  .reveal { position: relative; }
  .reveal-heading { animation: greeting-enter .6s ease-out both; }
  .result-card { position: relative; }
  .result-upper { overflow: hidden; isolation: isolate; }
  .result-upper > *:not(.blessing-burst) { position: relative; z-index: 1; }
  .fortune-scroll-mark { display: block; font: 700 13px var(--serif); letter-spacing: .34em; margin: 16px auto 0; color: #be9a5f; }
  .result-upper h2 { animation: fortune-unfurl .9s cubic-bezier(.18,.75,.25,1.2) both; }
  .result-card.celebration .result-upper { background: radial-gradient(circle at 50% 26%,#fff8dc,#fff7ea 58%,#f9ebd6); }
  .result-card.celebration .result-upper h2 { text-shadow: 0 4px 0 #eadfc4,0 0 18px #d6bc773b; }
  .result-card.celebration .result-upper::after { content: ''; position: absolute; inset: 0; z-index: 0; pointer-events: none; background: repeating-conic-gradient(from -5deg at 50% 38%,#d3b26310 0deg 13deg,transparent 13deg 28deg); animation: fortune-glow 10s linear infinite; }
  .special-message { color: #aa823f; font: 700 12px var(--serif); letter-spacing: .13em; margin: 22px 0 0; }
  .blessing-burst { pointer-events: none; position: absolute; inset: 0; z-index: 2; overflow: hidden; }
  .blessing-burst i { position: absolute; top: 43%; left: 50%; width: 9px; height: 15px; background: #d8aa51; border-radius: 2px; --burst-angle: calc(var(--index) * 20deg); opacity: 0; animation: blessing-burst 1.45s cubic-bezier(.16,.64,.33,1) both; animation-delay: calc(var(--index) * 35ms); }
  .blessing-burst i:nth-child(3n) { background: #d45d50; width: 6px; height: 10px; }
  .blessing-burst i:nth-child(4n) { background: #fff2be; border: 1px solid #c59b65; width: 6px; height: 6px; }
  .again-button { border: 0; background: transparent; color: #946947; font-size: 12px; padding: 10px; }
  .again-button:hover { text-decoration: underline; }
  .tie-section { margin: 0 auto 20px; padding: 18px; width: min(92%,520px); text-align: center; border-top: 1px dashed #d8c9b4; color: #827363; font-size: 12px; }
  .tie-section p { margin: 0 0 11px; }
  .tie-button { border: 1px solid #bb8a88; background: #fff4ee; color: #ab4a43; border-radius: 999px; padding: 9px 22px; font-weight: 700; font-size: 12px; }
  .tie-button:hover { background: #f8e8df; }
  .tie-done { margin: 10px 0 0 !important; font: 700 12px var(--serif); color: #56806d; }
  @keyframes ceremonial-shake { 0%,100% { transform: translateX(-50%) rotate(-11deg) translateY(0); } 50% { transform: translateX(-50%) rotate(12deg) translateY(-8px); } }
  @keyframes stick-left { from { transform: rotate(-12deg) translateY(0); } to { transform: rotate(-24deg) translateY(-18px); } }
  @keyframes stick-center { from { transform: rotate(4deg) translateY(0); } to { transform: rotate(-2deg) translateY(-25px); } }
  @keyframes stick-right { from { transform: rotate(20deg) translateY(0); } to { transform: rotate(31deg) translateY(-12px); } }
  @keyframes ritual-light { 0% { transform: rotate(var(--light-angle)) translateY(0) scale(.4); opacity: 0; } 30% { opacity: 1; } 100% { transform: rotate(var(--light-angle)) translateY(-215px) scale(1.1); opacity: 0; } }
  @keyframes moon-awaken { from { filter: drop-shadow(0 0 35px #dfc68a55); } to { filter: drop-shadow(0 0 78px #ffdc87b0); } }
  @keyframes fortune-unfurl { from { opacity: 0; transform: translateY(22px) scale(.8); filter: blur(5px); } to { opacity: 1; transform: translateY(0) scale(1); filter: blur(0); } }
  @keyframes greeting-enter { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: translateY(0); } }
  @keyframes fortune-glow { to { transform: rotate(360deg); } }
  @keyframes blessing-burst { 0% { opacity: 0; transform: rotate(var(--burst-angle)) translateY(0) scale(.5); } 28% { opacity: 1; } 100% { opacity: 0; transform: rotate(var(--burst-angle)) translateY(-195px) scale(1) rotate(100deg); } }
  @media(max-width:480px) {
    .ritual-caption { left: 18px; right: 18px; bottom: 47px; padding: 12px 9px; }
    .ritual-line { font-size: 12px; }
    .fortune-scroll-mark { font-size: 11px; }
  }
  @media(prefers-reduced-motion:reduce) {
    .stage.drawing .omikuji-art, .stage.drawing .fortune-stick, .stage.awakening .moon,
    .ritual-lights i, .blessing-burst i, .result-card.celebration .result-upper::after,
    .result-upper h2, .reveal-heading, .result-card { animation: none !important; transition: none !important; }
  }

</style>
