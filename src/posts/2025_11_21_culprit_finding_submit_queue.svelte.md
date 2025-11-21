---
title: 'Culprit-finding submit queue'
description: "High-throughput submit queues via group testing culprit finding"
author: 'Paul Hobbs'
date: '2025-11-21'
published: true
---

<script>
  let hoveredCL = null;
  let numCLs = 16;
  let numBatches = 8;
  const allCLs = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz".split('');

  $: cls = allCLs.slice(0, numCLs);

  let culprits = new Set(['C']);
  
  // Sparse Batches
  $: batches = (() => {
      const b = Array.from({ length: numBatches }, (_, i) => ({ id: i + 1, cls: [] }));
      const k = Math.min(3, numBatches/2);
      const used = new Set();

      cls.forEach(c => {
          let indices = new Set();
          let attempts = 0;
          let key = '';

          do {
              indices.clear();
              let pickAttempts = 0;
              while(indices.size < k && pickAttempts < 100) {
                  indices.add(Math.floor(Math.random() * numBatches));
                  pickAttempts++;
              }
              key = Array.from(indices).sort((x, y) => x - y).join(',');
              attempts++;
          } while (used.has(key) && attempts < 100);

          used.add(key);
          indices.forEach(i => b[i].cls.push(c));
      });
      return b;
  })();

  // Cleanup culprits
  $: {
      const valid = new Set(cls);
      let dirty = false;
      const next = new Set(culprits);
      for (const c of culprits) {
          if (!valid.has(c)) {
              next.delete(c);
              dirty = true;
          }
      }
      if (dirty) culprits = next;
  }

  // Reactive helpers
  const toggleCulprit = (c) => {
      const newCulprits = new Set(culprits);
      if (newCulprits.has(c)) {
          newCulprits.delete(c);
      } else {
          newCulprits.add(c);
      }
      culprits = newCulprits;
  };

  $: traditionalFail = cls.some(c => culprits.has(c));

  $: getBatchStatus = (batchCls) => batchCls.some(c => culprits.has(c)) ? 'fail' : 'pass';
  
  $: getScore = (c) => {
      const myBatches = batches.filter(b => b.cls.includes(c));
      if (myBatches.length === 0) return 0;
      const failCount = myBatches.filter(b => b.cls.some(bc => culprits.has(bc))).length;
      return (failCount / myBatches.length) * 100;
  };

  $: isDimmed = (batchCls) => {
      if (!hoveredCL) return false;
      return !batchCls.includes(hoveredCL);
  };
  
  $: isHighlighted = (batchCls) => {
      if (!hoveredCL) return false;
      return batchCls.includes(hoveredCL);
  };
</script>

<style>
    /* Styles adapted for the blog post */
    .container {
        display: grid;
        grid-template-columns: 1fr;
        gap: 40px;
        width: 100%;
        margin: 2rem 0;
    }
    
    @media (min-width: 768px) {
        .container {
            grid-template-columns: 1fr 1fr;
        }
    }

    .panel {
        background-color: #ffffff;
        border-radius: 12px;
        padding: 20px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        border: 1px solid #e5e7eb;
        display: flex;
        flex-direction: column;
        align-items: center;
        color: #1f2937;
    }

    .panel h3 {
        color: #2563eb;
        margin-top: 0;
        border-bottom: 2px solid #e5e7eb;
        padding-bottom: 10px;
        width: 100%;
        text-align: center;
        font-size: 1.2rem;
    }

    .queue-vis {
        display: flex;
        flex-direction: column;
        align-items: center;
        width: 100%;
        margin: 20px 0;
    }

    .cl-row {
        display: flex;
        gap: 10px;
        margin-bottom: 20px;
        justify-content: center;
        flex-wrap: wrap;
    }

    .cl {
        width: 32px;
        height: 32px;
        border-radius: 50%;
        background-color: #e5e7eb;
        display: flex;
        align-items: center;
        justify-content: center;
        font-weight: bold;
        font-size: 0.85rem;
        color: #1f2937;
        cursor: default;
        transition: transform 0.2s, opacity 0.2s;
        user-select: none;
    }
    
    .cl.bad {
        background-color: #ef4444;
        box-shadow: 0 0 10px rgba(239, 68, 68, 0.5);
        color: white;
        z-index: 2;
    }

    .cl.good {
        background-color: #86efac;
        color: #064e3b;
    }

    .arrow {
        margin: 10px 0;
        color: #6b7280;
        font-size: 0.9rem;
        text-align: center;
    }

    /* Traditional */
    .batch-box {
        border: 3px dashed #9ca3af;
        padding: 15px;
        border-radius: 8px;
        margin-top: 10px;
        width: 80%;
        text-align: center;
    }

    .batch-box.fail {
        border-color: #ef4444;
        background-color: #fef2f2;
    }

    .result-text {
        font-size: 1.2rem;
        margin-top: 15px;
        font-weight: bold;
    }
    .fail-text { color: #ef4444; }
    .pass-text { color: #22c55e; }

    /* Sparse */
    .matrix-container {
        display: flex;
        flex-wrap: wrap;
        gap: 10px;
        justify-content: center;
        margin-top: 10px;
        width: 100%;
    }

    .minibatch {
        border: 2px solid #d1d5db;
        border-radius: 6px;
        padding: 5px;
        width: 100px;
        min-height: 60px;
        display: flex;
        justify-content: center;
        align-items: center;
        align-content: center;
        gap: 3px;
        flex-wrap: wrap;
        transition: all 0.3s;
    }

    .minibatch.dimmed {
        opacity: 0.2;
    }
    
    .minibatch.highlight {
        transform: scale(1.1);
        z-index: 10;
        box-shadow: 0 0 15px rgba(0,0,0,0.2);
    }

    .mini-cl {
        width: 12px;
        height: 12px;
        border-radius: 50%;
        font-size: 7px;
        display: flex;
        align-items: center;
        justify-content: center;
        color: white;
    }
    .mini-cl.bad { background-color: #ef4444; }
    .mini-cl.good { background-color: #86efac; color: #064e3b; }

    .minibatch.fail {
        border-color: #ef4444;
        background-color: #fef2f2;
    }
    
    .batch-box.pass {
        border-color: #22c55e;
        background-color: #f0fdf4;
    }

    .minibatch.pass {
        border-color: #22c55e;
        background-color: #f0fdf4;
    }

    .scores {
        width: 100%;
        margin-top: 20px;
        font-family: monospace;
        font-size: 0.8rem;
        max-height: 500px;
        overflow-y: auto;
        border: 1px solid #e5e7eb;
        border-radius: 6px;
        padding: 0;
    }
    
    .score-row {
        display: flex;
        justify-content: space-between;
        padding: 3px 10px;
        align-items: center;
        border-bottom: 1px solid #e5e7eb;
    }

    .score-bar-container {
        width: 80px;
        background: #e5e7eb;
        height: 10px;
        align-self: center;
        border-radius: 5px;
        overflow: hidden;
        margin: 0 10px;
    }

    .score-bar {
        height: 100%;
        background: #3b82f6;
        transition: width 0.5s;
    }

    .score-bar.high { background: #ef4444; }
    
    .false-reject { color: #f59e0b; font-weight: bold; }
    
    .legend {
        display: flex;
        gap: 15px;
        margin: 20px 0;
        font-size: 0.85rem;
        justify-content: center;
        flex-wrap: wrap;
    }
    .legend-item { display: flex; align-items: center; gap: 6px; color: #4b5563; }
    .dot { width: 10px; height: 10px; border-radius: 50%; }
    .dot-red { background: #ef4444; }
    .dot-green { background: #86efac; }
    .dot-blue { background: #3b82f6; }
    
    .description {
        margin-top: 20px;
        line-height: 1.6;
        color: #4b5563;
        font-size: 0.9rem;
    }
    
    .controls {
        display: flex;
        gap: 20px;
        justify-content: center;
        margin-bottom: 20px;
        background: #f3f4f6;
        padding: 15px;
        border-radius: 8px;
        flex-wrap: wrap;
    }
    .control-group {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 5px;
        font-size: 0.9rem;
        color: #374151;
    }
    input[type=range] {
        cursor: pointer;
    }
</style>

I've been spending some time modeling submit queues recently. If you've worked on a large enough team, you know the pain: you merge a PR, go to grab coffee, and come back to find the build is broken. The "tree is red," and nobody can merge until the culprit is found and reverted.

As the rate of changes increases, we usually start batching them together to save resources. But that introduces a new problem: when a batch fails, which change caused it?

## The "Batch & Pray" Dilemma

The traditional approach is simple: group a bunch of changes, run the tests, and if they pass, merge them all. This works great when everything is green.

But when a test fails, you're stuck. You know *one* of these ten changes broke the build, but you don't know which one. The system usually has to reject the whole batch or stop everything to run a binary search (bisect) to find the bad commit. This kills throughput exactly when you need it most—when the queue is backing up.

## Sparse Bernoulli Grouping

I started looking into "Group Testing" algorithms—the kind used in medical testing to screen large populations with few tests. The specific flavor that fits here is **Sparse Bernoulli** culprit finding.

The idea is to assign each change to multiple, overlapping small batches (minibatches) instead of one big one.

1.  **Distribute**: We assign each change to $k$ random minibatches.
2.  **Test**: We run all these minibatches in parallel.
3.  **Score**: If a change is bad, it poisons every batch it touches. If a change is good, it might get unlucky and share a batch with a culprit, but it should also appear in passing batches (unless it's *really* unlucky).

In the simulation code I've been playing with (written in Go), I use a few heuristics to tune this:

*   **Dynamic Batch Count ($T$)**: The number of minibatches scales logarithmically with the queue size ($N$). Specifically, $T \approx 10 \log_{10} N$. This keeps the resource usage reasonable even as the queue grows.
*   **Replication Factor ($k$)**: We calculate how many batches each change should be in based on the expected number of culprits. The formula is roughly $k \approx T / (d + 1)$, where $$d$$ is the expected number of bad changes (usually around 2).
*   **Scoring**: We calculate a "Suspicion Score" for each change. It's simply the percentage of that change's batches that failed. If the score is above a threshold (I'm using 0.75), we reject the change.

There's also a concept of **Weighted Scoring**. If we know certain tests are flaky, we can trust their failures less. In the Go implementation, if a batch fails, we weight that failure by the reliability of the test that failed. A 99% reliable test failing carries more weight than a 50% reliable test.

## Interactive Simulation

Click on any change node to toggle it as a culprit. Hover over the changes in the "Sparse Bernoulli" panel below to see how they map to batches. Notice how **culprits** cause every batch they touch to fail.

<div class="legend">
    <div class="legend-item"><div class="dot dot-green"></div>Good Change</div>
    <div class="legend-item"><div class="dot dot-red"></div>Bad Change (Culprit)</div>
    <div class="legend-item"><div class="dot dot-blue"></div>Suspicion Score</div>
</div>

<div class="controls">
    <div class="control-group">
        <label for="cl-slider">Changes: {numCLs}</label>
        <input id="cl-slider" type="range" min="5" max="50" bind:value={numCLs} />
    </div>
    <div class="control-group">
        <label for="batch-slider">Minibatches: {numBatches}</label>
        <input id="batch-slider" type="range" min="1" max="20" bind:value={numBatches} />
    </div>
</div>

<div class="container">
    <!-- Traditional Approach -->
    <div class="panel">
        <h3>Traditional: "Batch & Pray"</h3>
        
        <div class="queue-vis">
            <div class="cl-row">
                {#each cls as c}
                    <div class="cl {culprits.has(c) ? 'bad' : 'good'}">{c}</div>
                {/each}
            </div>

            <div class="arrow">⬇ Group into Single Batch ⬇</div>

            <div class="batch-box {traditionalFail ? 'fail' : 'pass'}">
                <strong>Batch #1</strong><br>
                [{cls.join(', ')}]
            </div>

            <div class="result-text {traditionalFail ? 'fail-text' : 'pass-text'}">
                {traditionalFail ? 'BATCH FAILED' : 'BATCH PASSED'}
            </div>
        </div>

        <div class="description">
            <p><strong>Logic:</strong> Combine all pending changes into one big test batch.</p>
            <p><strong>The Problem:</strong> If <em>any</em> change is bad, the <em>entire batch fails</em>.</p>
            <p><strong>Consequence:</strong> The system rejects everyone (innocent changes included) or halts to bisect.</p>
        </div>
    </div>

    <!-- Sparse Bernoulli Approach -->
    <div class="panel">
        <h3>New: Sparse Bernoulli Grouping</h3>
        
        <div class="queue-vis">
            <div class="cl-row">
                {#each cls as c}
                    <!-- svelte-ignore a11y-mouse-events-have-key-events -->
                    <div 
                        class="cl {culprits.has(c) ? 'bad' : 'good'}"
                        on:mouseenter={() => hoveredCL = c}
                        on:mouseleave={() => hoveredCL = null}
                        style="cursor: pointer;"
                        on:click={() => toggleCulprit(c)}
                        on:keydown={(e) => e.key === 'Enter' && toggleCulprit(c)}
                        role="button"
                        tabindex="0"
                    >
                        {c}
                    </div>
                {/each}
            </div>

            <div class="arrow">⬇ Distribute into Overlapping Minibatches ⬇</div>

            <div class="matrix-container">
                {#each batches as batch}
                    <div 
                        class="minibatch {getBatchStatus(batch.cls)} {isDimmed(batch.cls) ? 'dimmed' : ''} {isHighlighted(batch.cls) ? 'highlight' : ''}"
                    >
                        {#each batch.cls as c}
                            <div class="mini-cl {culprits.has(c) ? 'bad' : 'good'}">{c}</div>
                        {/each}
                    </div>
                {/each}
            </div>

            <div class="scores">
                {#each cls as c}
                    {@const pct = getScore(c)}
                    <div class="score-row">
                        <span>{c}</span>
                        <div class="score-bar-container">
                            <div class="score-bar {pct > 70 ? 'high' : ''}" style="width: {pct}%"></div>
                        </div>
                        <span>
                            {Math.round(pct)}% 
                            {#if pct > 70}
                                {#if culprits.has(c)}
                                    (REJECT)
                                {:else}
                                    <span class="false-reject">(FALSE REJECT)</span>
                                {/if}
                            {/if}
                        </span>
                    </div>
                {/each}
            </div>
        </div>

        <div class="description">
            <p><strong>Logic:</strong> Assign each change to multiple random small batches.</p>
            <p><strong>The Magic:</strong> Culprits fail every batch they touch. Innocent changes might share a failing batch with a culprit, but they also exist in <em>passing</em> batches (unless unlucky).</p>
        </div>
    </div>
</div>

It's a probabilistic approach, so it's not magic, but the math suggests we can identify culprits without halting the entire line.
