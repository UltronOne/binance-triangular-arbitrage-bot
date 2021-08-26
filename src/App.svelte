<script>
  import { onMount } from "svelte";
  import dayjs from "dayjs";
  import buildPairs from "./lib/pairs";
  import buildProfitFunc from "./lib/profit";

  import socketSubscribe from "./lib/socket";

  import ResultsTable from "./winners-table.svelte";
  import DetailModal from "./detail.svelte";

  let modalDetail = false;
  let contentModal = {};

  const showPopupLong = (e) => {
    modalDetail = true;
    contentModal = e;
  };

  const info = {
    pairslen: 0,
    msgcount: 0,
    cycleschecked: 0,
    cyclescheckedPerSecond: 0,
    currentStatus: "",
  };
  let oldWinners = null;
  let winners = [];
  let tops = [];

  let checkProfit;

  onMount(async () => {
    const socket = await socketSubscribe();
    const { hashMarket, pairs: allPairs } = await buildPairs();

    info.currentStatus = "Connecting to binance";

    info.pairslen = allPairs.length;
    info.socket = socket;
    setInterval(() => {
      info.msgcount = socket.msgcount;
      socket.msgcount = 0;
      info.cyclescheckedPerSecond = info.cycleschecked;
      info.cycleschecked = 0;
    }, 250);

    checkProfit = await buildProfitFunc();

    function tick() {
      if (info.currentStatus === "Connecting to binance") {
        if (info.msgcount > 0) {
          info.currentStatus = "Working";
        }
      } else {
        info.currentStatus = "Working";
      }

      const startTime = +new Date();
      const uPair = socket.pairupdate;
      const markets = socket.m;

      socket.pairupdate = [];

      let pairsToTest = uPair.map((pair) => hashMarket[pair]).flat();
      pairsToTest = [...new Set(pairsToTest)]
        .map((id) => allPairs[id])
        .filter((e) => e);

      if (!pairsToTest.length) {
        setTimeout(tick, 50);
        return;
      }

      // voy con todos!
      // const pairsToTest = allPairs.filter(p => intersect(p, u).length)

      let ret;
      try {
        info.cycleschecked += pairsToTest.length;
        ret = checkProfit(pairsToTest, markets);

        ret.sort((a, b) => b.profit.sub(a.profit));

        const w = ret.filter((e) => e.profit.gt(1));

        if (w.length) {
          winners = w.filter((e) => e.chain[0] == "USDT");
          console.log(winners);
          winners.sort((a, b) => a.profit.cmp(b.profit) > 1);
          w.forEach((e) => {
            e.time = dayjs().format();

            for (let k = 0; k < e.length - 1; k += 1) {
              delete socket.m[e[k] + "/" + e[k + 1]];
              delete socket.m[e[k + 1] + "/" + e[k]];
            }
          });
        }
      } catch (err) {
        console.error(err);
        console.log(pairsToTest);
      }

      console.log("cycle", +new Date() - startTime, "ms");
      if (winners.length) {
        info.currentStatus = "Found profitable cycles, halt for 5 seconds";

        // halt 30 seconds
        setTimeout(() => {
          info.currentStatus = "Working";
          oldWinners = winners;
          winners = [];
          tick();
        }, 1000 * 5);
        return;
      }

      tops = ret.slice(0, 10);

      setTimeout(tick, 50);
    }
    setTimeout(tick, 1000);
    return () => {};
  });
</script>

<main>
  <h1>
    <a href="https://accounts.binance.com/de/register?ref=38018852">Binance</a> triangular
    arbitrage finder in real time
  </h1>

  Total combinations: {info.pairslen} pairs<br />
  Socket updates per second: {info.msgcount}<br />
  Cycles checked per second: {info.cyclescheckedPerSecond}<br />

  <big>
    Current status<br />
    <b>{info.currentStatus}</b><br />
  </big>

  <h2>WINNERS</h2>
  {#if !winners.length}
    <b>No winner pair yet</b>
  {:else}
    <ResultsTable results={winners} />
  {/if}

  {#if tops.length}
    <h2>Pairs</h2>
    <table class="styled-table pairs-table">
      <thead>
        <th>Pairs cycle</th>
        <th>Profit (BNB Fess included)</th>
      </thead>
      <tbody>
        {#each tops as t}
          <tr>
            <td style="min-width: 235px;">
              {@html t.chain.join(" &rarr; ")}
            </td>
            <td>
              <pre
                style="cursor:pointer"
                on:click={() =>
                  showPopupLong(
                    t
                  )}>{t.profit.sub(1).mul(100).toFixed( 4 )}% &#9432;</pre>
            </td>
          </tr>
        {/each}
      </tbody>
    </table>
  {/if}
  <blockquote>
    Trades profit are calculate on a base budget of 100USD and a 0.0075% fee
  </blockquote>

  {#if oldWinners && oldWinners.length}
    <h2>PREV CYCLE WINNERS</h2>
    <ResultsTable results={oldWinners} />
  {/if}
</main>
{#if modalDetail}
  <DetailModal on:close={() => (modalDetail = false)}>
    <ResultsTable results={[contentModal]} />
  </DetailModal>
{/if}

<style>
  table {
    margin: auto;
  }
  main {
    text-align: center;
    padding: 1em;
    max-width: 240px;
    margin: 0 auto;
  }

  h1 {
    color: #0fb948;
    text-transform: uppercase;
    font-size: 2em;
    font-weight: 400;
  }

  @media (min-width: 640px) {
    main {
      max-width: none;
    }
  }

  .pairs-table {
    max-width: 320px;
    margin: 30px auto;
  }
  .pairs-table td {
    padding: 5px 15px;
  }
</style>
