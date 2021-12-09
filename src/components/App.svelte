<script lang="ts">
  import Visual from "./Visual.svelte";
  import InputRange from "./InputRange.svelte";
  import Checkbox from "./Checkbox.svelte";
  import ParticleSelector from "./ParticleSelector.svelte";
  import type { Particle } from "../types";
  import { writable } from "svelte/store";
  import TRANSLATIONS from "../translations";

  function getTranslation(lang: string, key: string) {
    const phrase: { [key: string]: string } = TRANSLATIONS[key];
    return Object.keys(phrase).includes(lang) ? phrase[lang] : phrase["en"];
  }

  let width: number = window.innerWidth;
  let height: number = window.innerHeight;

  let lang = new URLSearchParams(location.search).get("lang") || "en";

  const COLORS = [
    "#fa1414",
    "#c88c64",
    "#50aa8c",
    "#0096e6",
    "#0a14e6",
    "#8200c8",
    "#fa96d2",
    "#828282",
    "green",
    "rgb(200, 200, 200)",
  ];

  let showSettings = true;

  let r = Number(new URLSearchParams(location.search).get("radius")) || 5;
  let nodeCount =
    Number(new URLSearchParams(location.search).get("count")) || 750;
  let MAX_DIST =
    Number(new URLSearchParams(location.search).get("max_dist")) || 100;
  let speedMultiplier =
    Number(new URLSearchParams(location.search).get("temperature")) || 4;
  let BORDER_X = 10;
  let BORDER_Y = 10;
  let selectedId = 0;

  let friction =
    Number(new URLSearchParams(location.search).get("friction")) || 0.98;

  let drawConnections = true;
  let changeFormBySpeed = true;
  let displacementMultiplier = 1;

  const maxDist2 = MAX_DIST * MAX_DIST;
  let fw = width / MAX_DIST + 1;
  let fh = height / MAX_DIST + 1;

  let RULE_SIZE = 3;
  let MAX_LINKS: number[] = [];
  let COUPLINGS: number[][] = [];
  let MAX_LINKS_PER_COLOR: number[][] = [];

  let links: { a: Particle; b: Particle }[] = [];
  let fields: Particle[][][] = [];

  let settingRuleSize = writable(3);
  let simulationsPerFrame = 5;

  let linkDistance = 1;

  let play = true;

  let connectBorders = false;

  if (location.hash) {
    let hash = location.hash.substring(1).split("-");
    let c = 0;
    $settingRuleSize = ~~hash[c];
    RULE_SIZE = $settingRuleSize;
    // settingRange.value = RULE_SIZE;
    // rangeValue.textContent = RULE_SIZE;
    startNew();
    c++;
    for (let i = 0; i < RULE_SIZE; i++) {
      MAX_LINKS[i] = (Number(hash[c]) >> (i * 2)) & 3;
    }
    c++;
    for (let i = 0; i < RULE_SIZE; i++) {
      for (let j = 0; j < RULE_SIZE; j++) {
        COUPLINGS[i][j] = ((Number(hash[c]) >> (j * 2)) & 3) - 1;
      }
      c++;
    }
    for (let i = 0; i < RULE_SIZE; i++) {
      for (let j = 0; j < RULE_SIZE; j++) {
        MAX_LINKS_PER_COLOR[i][j] = (Number(hash[c]) >> (j * 2)) & 3;
      }
      c++;
    }
    history.pushState(null, null, " ");
  } else {
    startNew();
  }

  requestAnimationFrame(update);

  function startNew() {
    selectedId = 0;
    generateRules();
    generateNodes();
  }

  function generateNodes() {
    links = [];
    fields = [];
    for (let i = 0; i < fw; i++) {
      fields.push([]);
      for (let j = 0; j < fh; j++) {
        fields[i].push([]);
      }
    }
    for (let i = 0; i < nodeCount; i++) {
      addParticle(
        Math.random() * width,
        Math.random() * height,
        ~~(Math.random() * RULE_SIZE)
      );
    }
  }

  function generateRules() {
    RULE_SIZE = $settingRuleSize;
    MAX_LINKS = [];
    COUPLINGS = [];
    MAX_LINKS_PER_COLOR = [];
    for (let i = 0; i < RULE_SIZE; i++) {
      MAX_LINKS.push(~~(Math.random() * 4));
      COUPLINGS.push([]);
      MAX_LINKS_PER_COLOR.push([]);
      for (let j = 0; j < RULE_SIZE; j++) {
        COUPLINGS[i].push(Math.floor(Math.random() * 3 - 1));
        MAX_LINKS_PER_COLOR[i].push(~~(Math.random() * 4));
      }
    }
    // console.log(MAX_LINKS_PER_COLOR);
  }

  function copyRules() {
    let hash = "";
    hash += RULE_SIZE;
    let links_hash = 0;
    for (let i = 0; i < RULE_SIZE; i++) {
      links_hash |= MAX_LINKS[i] << (i * 2);
    }
    hash += "-" + links_hash;
    for (let i = 0; i < RULE_SIZE; i++) {
      let row = 0;
      for (let j = 0; j < RULE_SIZE; j++) {
        row |= (COUPLINGS[i][j] + 1) << (j * 2);
      }
      hash += "-" + row;
    }
    for (let i = 0; i < RULE_SIZE; i++) {
      let row = 0;
      for (let j = 0; j < RULE_SIZE; j++) {
        row |= MAX_LINKS_PER_COLOR[i][j] << (j * 2);
      }
      hash += "-" + row;
    }
    hash = window.location.href + "#" + hash;
    let tempInput = document.createElement("input");
    tempInput.value = hash;
    document.body.appendChild(tempInput);
    tempInput.select();
    document.execCommand("copy");
    document.body.removeChild(tempInput);
  }

  function addParticle(x: number, y: number, type: number, name?: string) {
    let a: Particle = {
      x,
      y,
      lastX: x,
      lastY: y,
      type,
      sx: 0,
      sy: 0,
      bonds: [],
    };
    if (name) a.name = name;
    const fx = Math.floor(x / MAX_DIST);
    const fy = Math.floor(y / MAX_DIST);
    fields[fx][fy].push(a);
  }

  function removeFromArray<T>(array: T[], item: T) {
    array.splice(array.indexOf(item), 1);
  }

  function linkParticles(a: Particle, b: Particle) {
    a.bonds.push(b);
    b.bonds.push(a);
    links.push({ a, b });
  }

  function applyForce(a: Particle, b: Particle) {
    const dx = a.x - b.x;
    const dy = a.y - b.y;
    let dist2 = dx * dx + dy * dy;
    if (dist2 < maxDist2) {
      let da = COUPLINGS[a.type][b.type] / dist2;
      let db = COUPLINGS[b.type][a.type] / dist2;
      if (
        a.bonds.length < MAX_LINKS[a.type] &&
        b.bonds.length < MAX_LINKS[b.type]
      ) {
        if (dist2 < (maxDist2 / 4) * linkDistance) {
          if (a.bonds.indexOf(b) === -1 && b.bonds.indexOf(a) === -1) {
            let typeCountA = 0;
            a.bonds.forEach((bond) => {
              if (bond.type === b.type) typeCountA++;
            });
            let typeCountB = 0;
            b.bonds.forEach((bond) => {
              if (bond.type === a.type) typeCountB++;
            });
            if (
              typeCountA < MAX_LINKS_PER_COLOR[a.type][b.type] &&
              typeCountB < MAX_LINKS_PER_COLOR[b.type][a.type]
            ) {
              linkParticles(a, b);
            }
          }
        }
      } else {
        if (a.bonds.indexOf(b) === -1 && b.bonds.indexOf(a) === -1) {
          da = 1 / dist2;
          db = 1 / dist2;
        }
      }
      const angle = Math.atan2(a.y - b.y, a.x - b.x);
      if (dist2 < 1) dist2 = 1;
      if (dist2 < r * r * 4) {
        da = 1 / dist2;
        db = 1 / dist2;
      }
      a.sx += Math.cos(angle) * da * speedMultiplier;
      a.sy += Math.sin(angle) * da * speedMultiplier;
      b.sx -= Math.cos(angle) * db * speedMultiplier;
      b.sy -= Math.sin(angle) * db * speedMultiplier;
    }
  }

  function logic() {
    for (let i = 0; i < fw; i++) {
      for (let j = 0; j < fh; j++) {
        const field = fields[i][j];
        for (let k = 0; k < field.length; k++) {
          const a = field[k];
          a.lastX = a.x;
          a.lastY = a.y;
          a.x += a.sx;
          a.y += a.sy;
          if (connectBorders) {
            if (a.x < BORDER_X) {
              a.x = width - BORDER_X;
              a.sx * -1;
            } else if (a.x > width - BORDER_X) {
              a.x = BORDER_X;
              a.sx * -1;
            }
            if (a.y < BORDER_Y) {
              a.y = height - BORDER_Y;
              a.sy * -1;
            } else if (a.y > height - BORDER_Y) {
              a.y = BORDER_Y;
              a.sy * -1;
            }
          }
          a.sx *= friction;
          a.sy *= friction;
          const magnitude = Math.sqrt(a.sx * a.sx + a.sy * a.sy);
          if (magnitude > 1) {
            a.sx /= magnitude;
            a.sy /= magnitude;
          }
          if (!connectBorders) {
            if (a.x < BORDER_X) {
              a.sx += speedMultiplier * 0.05;
              if (a.x < 0) {
                a.x = -a.x;
                a.sx *= -0.5;
              }
            } else if (a.x > width - BORDER_X) {
              a.sx -= speedMultiplier * 0.05;
              if (a.x > width) {
                a.x = width * 2 - a.x;
                a.sx *= -0.5;
              }
            }
            if (a.y < BORDER_Y) {
              a.sy += speedMultiplier * 0.05;
              if (a.y < 0) {
                a.y = -a.y;
                a.sy *= -0.5;
              }
            } else if (a.y > height - BORDER_Y) {
              a.sy -= speedMultiplier * 0.05;
              if (a.y > height) {
                a.y = height * 2 - a.y;
                a.sy *= -0.5;
              }
            }
          }
        }
      }
    }
    for (let i = 0; i < links.length; i++) {
      const link = links[i];
      const a = link.a;
      const b = link.b;
      const dx = a.x - b.x;
      const dy = a.y - b.y;
      let dist2 = dx * dx + dy * dy;
      if (dist2 > (maxDist2 / 4) * linkDistance) {
        removeFromArray(a.bonds, b);
        removeFromArray(b.bonds, a);
        removeFromArray(links, link);
        i--;
      } else {
        if (dist2 > r * r * 4) {
          const angle = Math.atan2(a.y - b.y, a.x - b.x);
          const dA = -0.015;

          a.sx += Math.cos(angle) * dA * speedMultiplier;
          a.sy += Math.sin(angle) * dA * speedMultiplier;
          b.sx -= Math.cos(angle) * dA * speedMultiplier;
          b.sy -= Math.sin(angle) * dA * speedMultiplier;
        }
      }
    }
    for (let i = 0; i < fw; i++) {
      for (let j = 0; j < fh; j++) {
        const field = fields[i][j];
        for (let k = 0; k < field.length; k++) {
          const a = field[k];
          const fx = Math.floor(a.x / MAX_DIST);
          const fy = Math.floor(a.y / MAX_DIST);
          if (fx !== i || fy !== j) {
            removeFromArray(field, a);
            fields[fx][fy].push(a);
          }
        }
      }
    }
    for (let i = 0; i < fw; i++) {
      for (let j = 0; j < fh; j++) {
        const field = fields[i][j];
        for (let i1 = 0; i1 < field.length; i1++) {
          const a = field[i1];
          for (let j1 = i1 + 1; j1 < field.length; j1++) {
            const b = field[j1];
            applyForce(a, b);
          }
          if (i < fw - 1) {
            const iNext = i + 1;
            const field1 = fields[iNext][j];
            for (let j1 = 0; j1 < field1.length; j1++) {
              const b = field1[j1];
              applyForce(a, b);
            }
          }
          if (j < fh - 1) {
            const jNext = j + 1;
            const field1 = fields[i][jNext];
            for (let j1 = 0; j1 < field1.length; j1++) {
              const b = field1[j1];
              applyForce(a, b);
            }
            if (i < fw - 1) {
              const iNext = i + 1;
              const field2 = fields[iNext][jNext];
              for (let j1 = 0; j1 < field2.length; j1++) {
                const b = field2[j1];
                applyForce(a, b);
              }
            }
          }
        }
      }
    }
  }

  function update() {
    if (play) {
      for (let i = 0; i < simulationsPerFrame; i++) logic();
    }
    fields = [...fields];
    window.requestAnimationFrame(update);
  }

  let controlsHidden = true;

  let newParticleName = "";

  $: addParticlesAmount = COLORS.map(() => 0);

  // $: {
  //   const searchParams = new URLSearchParams(
  //     `?temperature=${speedMultiplier}&friction=${friction}&radius=${r}`
  //   );
  //   const newurl =
  //     window.location.protocol +
  //     "//" +
  //     window.location.host +
  //     window.location.pathname +
  //     "?" +
  //     searchParams.toString();
  //   window.history.replaceState({ path: newurl }, "", newurl);
  // }
</script>

<svelte:window
  bind:innerWidth={width}
  bind:innerHeight={height}
  on:mousemove={({ clientX }) => {
    controlsHidden = clientX > 500;
  }}
/>
<main>
  <Visual
    {width}
    {height}
    {fields}
    {links}
    {fw}
    {fh}
    {r}
    {drawConnections}
    {changeFormBySpeed}
    {displacementMultiplier}
    colors={COLORS}
    on:click={(e) => {
      addParticle(
        e.clientX + Math.random() - 0.5,
        e.clientY + Math.random() - 0.5,
        selectedId,
        newParticleName
      );
      // logic();
    }}
  />
  <div
    class="controls"
    class:controls_opened={showSettings}
    class:dispay-none={controlsHidden}
  >
    {#if showSettings}
      <!-- <div class="controls__col"> -->
      <div class="controls-block">
        <div class="buttons-row">
          <button on:click={() => (showSettings = false)}>
            {getTranslation(lang, "hideSettings")}
          </button>
          <button on:click={copyRules}>
            {getTranslation(lang, "copyLink")}
          </button>
          <button
            on:click={() => {
              play = !play;
            }}
          >
            {play ? "pause" : "play"}
          </button>
        </div>
      </div>
      <div class="controls-block">
        <h2 class="controls-block__title">
          {getTranslation(lang, "currentWorldSettings")}
        </h2>
        <InputRange
          name={"link distance"}
          min={0.001}
          step={0.001}
          max={MAX_DIST}
          bind:value={linkDistance}
        />
        <InputRange
          name={getTranslation(lang, "simulationsPerFrame")}
          min={1}
          max={100}
          bind:value={simulationsPerFrame}
        />
        <InputRange
          name={getTranslation(lang, "temperature")}
          min={0.1}
          max={40}
          step={0.1}
          bind:value={speedMultiplier}
        />
        <InputRange
          name={getTranslation(lang, "friction")}
          min={0}
          max={1}
          step={0.01}
          bind:value={friction}
        />
        <InputRange
          name={getTranslation(lang, "particleRadius")}
          min={3}
          max={10}
          step={0.01}
          bind:value={r}
        />
        <InputRange
          name={"borders x"}
          min={10}
          max={window.innerWidth / 2}
          step={0.01}
          bind:value={BORDER_X}
        />
        <InputRange
          name={"borders y"}
          min={10}
          max={window.innerHeight / 2}
          step={0.01}
          bind:value={BORDER_Y}
        />
        <Checkbox title={"Connect borders"} bind:checked={connectBorders} />
        <div class="buttons-row">
          <button
            on:click={() => {
              fields = [];
              for (let i = 0; i < fw; i++) {
                fields.push([]);
                for (let j = 0; j < fh; j++) {
                  fields[i].push([]);
                }
              }
              links = [];
            }}
          >
            {getTranslation(lang, "killAllParticles")}
          </button>
        </div>
      </div>
      <div class="controls-block">
        <h2 class="controls-block__title">MAX_LINKS</h2>
        {#each Object.entries(MAX_LINKS) as [k, v]}
          <div class="max-links-input">
            <span
              class="table-label"
              style={`background-color: ${COLORS[k]}; padding: 0 5px`}
            >
              {k}
            </span>
            <input
              type="number"
              value={v}
              min="0"
              max="3"
              on:input={(e) => {
                // @ts-ignore
                MAX_LINKS[k] = +e.target.value;
              }}
            />
          </div>
        {/each}
      </div>
      <div class="controls-block">
        <h2 class="controls-block__title">COUPLINGS</h2>
        <table>
          <tbody>
            <tr class="couplings-row">
              <th />
              {#each Object.entries(COLORS.slice(0, RULE_SIZE)) as [i, color]}
                <th class="table-label" style={`background-color: ${COLORS[i]}`}
                  >{i}</th
                >
              {/each}
            </tr>
            {#each Object.entries(COUPLINGS) as [y, arr]}
              <tr class="couplings-row">
                <td class="table-label" style={`background-color: ${COLORS[y]}`}
                  >{y}</td
                >
                {#each Object.entries(arr) as [x, val]}
                  <td>
                    <input
                      type="number"
                      value={val}
                      min={-1}
                      max={1}
                      on:input={(e) => {
                        // @ts-ignore
                        COUPLINGS[y][x] = e.target.value;
                      }}
                    />
                  </td>
                {/each}
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
      <!-- MAX_LINKS_PER_COLOR -->
      <div class="controls-block">
        <h2 class="controls-block__title">MAX_LINKS_PER_COLOR</h2>
        <table>
          <tbody>
            <tr class="couplings-row">
              <th />
              {#each Object.entries(COLORS.slice(0, RULE_SIZE)) as [i, color]}
                <th class="table-label" style={`background-color: ${COLORS[i]}`}
                  >{i}</th
                >
              {/each}
            </tr>
            {#each Object.entries(MAX_LINKS_PER_COLOR) as [y, arr]}
              <tr class="couplings-row">
                <td class="table-label" style={`background-color: ${COLORS[y]}`}
                  >{y}</td
                >
                {#each Object.entries(arr) as [x, val]}
                  <td>
                    <input
                      type="number"
                      value={val}
                      min={0}
                      max={3}
                      on:input={(e) => {
                        // @ts-ignore
                        MAX_LINKS_PER_COLOR[y][x] = e.target.value;
                      }}
                    />
                  </td>
                {/each}
              </tr>
            {/each}
          </tbody>
        </table>
      </div>
      <div class="controls-block">
        <h2 class="controls-block__title">Add particles</h2>
        <div class="compact-inputs">
          {#each Object.entries(COLORS.slice(0, RULE_SIZE)) as [i, color]}
            <div class="max-links-input">
              <span
                class="table-label"
                style={`background-color: ${color}; padding: 0 5px`}
              >
                {i}
              </span>
              <input
                type="number"
                value={addParticlesAmount[i]}
                min="0"
                max="3"
                on:input={(e) => {
                  // @ts-ignore
                  addParticlesAmount[i] = +e.target.value || 0;
                }}
              />
            </div>
          {/each}
        </div>
        <div class="buttons-row">
          <button
            on:click={() => {
              addParticlesAmount.slice(0, RULE_SIZE).forEach((amount, type) => {
                for (let i = 0; i < amount; i++) {
                  addParticle(
                    Math.random() * width,
                    Math.random() * height,
                    type
                  );
                }
              });
            }}
          >
            Add particles
          </button>
        </div>
      </div>
      <div class="controls-block">
        <h2 class="controls-block__title">
          {getTranslation(lang, "newWorldSettings")}
        </h2>
        <InputRange
          name={getTranslation(lang, "particleTypesAmount")}
          bind:value={$settingRuleSize}
          min={1}
          max={COLORS.length}
        />
        <InputRange
          name={getTranslation(lang, "particleCount")}
          bind:value={nodeCount}
          min={0}
          max={5000}
        />
        <div class="buttons-row">
          <button on:click={startNew}>
            {getTranslation(lang, "createNewWorld")}
          </button>
        </div>
      </div>
      <div class="controls-block">
        <h2 class="controls-block__title">
          {getTranslation(lang, "particleBrush")}
        </h2>
        <ParticleSelector
          colors={COLORS.slice(0, COUPLINGS.length)}
          bind:selectedId
        />
        <input
          class="particle-name-input"
          type="text"
          placeholder="Enter new particle name"
          bind:value={newParticleName}
        />
      </div>

      <div class="controls-block">
        <h2 class="controls-block__title">
          {getTranslation(lang, "graphicalSettings")}
        </h2>
        <Checkbox
          title={getTranslation(lang, "drawConnections")}
          bind:checked={drawConnections}
        />
        <Checkbox
          title={getTranslation(lang, "changeFormBySpeed")}
          bind:checked={changeFormBySpeed}
        />
        {#if changeFormBySpeed}
          <InputRange
            name={getTranslation(lang, "displacementMultiplier")}
            min={1}
            max={10}
            step={1}
            bind:value={displacementMultiplier}
          />
        {/if}
      </div>
      <!-- </div> -->
      <!-- <div class="controls__col">
        <div class="controls-block">
          <h2 class="controls-block__title">View settings</h2>
          <div class="buttons-row">
            <button on:click={startNew}>Show connections</button>
          </div>
        </div>
      </div> -->
    {:else}
      <button on:click={() => (showSettings = true)}>
        {getTranslation(lang, "showSettings")}
      </button>
    {/if}
  </div>
</main>

<style>
  button {
    padding: 8px 15px;
    color: inherit;
    font-size: 13px;
    /* white-space: nowrap; */
    /* overflow: hidden;
    text-overflow: ellipsis; */
  }
  .controls {
    overflow-y: scroll;
    box-shadow: 0 0 10px 0 black;
    /* display: flex; */
    /* flex-direction: row; */
    position: fixed;
    /* width: 300px; */
    max-height: calc(100% - 40px);
    left: 20px;
    top: 20px;
    background-color: rgba(255, 255, 255, 0.5);
    /* padding: 10px; */
    border-radius: 5px;
    backdrop-filter: blur(8px);
    -webkit-backdrop-filter: blur(8px);
    -ms-overflow-style: none; /* IE and Edge */
    scrollbar-width: none; /* Firefox */
    transition-duration: 0.2s;
  }
  .controls::-webkit-scrollbar {
    display: none;
  }
  .controls_opened {
    padding: 10px;
    width: 300px;
    border-radius: 10px;
  }
  .controls__col {
    min-width: 300px;
    max-width: 300px;
  }
  .controls__col + .controls__col {
    /* min-width: 300px; */
    margin-left: 10px;
  }
  .controls-block > * {
    margin-bottom: 0;
  }
  .controls-block:not(:last-child) {
    margin-bottom: 20px;
  }
  .controls-block__title {
    margin: 0;
    margin-bottom: 5px;
    text-transform: uppercase;
    font-size: 100%;
  }
  .buttons-row {
    display: flex;
  }
  .buttons-row button {
    width: 100%;
  }
  @supports (not (backdrop-filter: blur())) and
    (not (-webkit-backdrop-filter: blur())) {
    .controls {
      background-color: rgba(150, 150, 150, 0.95);
    }
  }
</style>
