<script context="module">
  import SystemRequirements from '../services/SystemRequirements.js';
  SystemRequirements.addJS('CSS Typed OM', () => !!(CSS && CSS.number));
</script>

<script>
  import { onMount, onDestroy } from 'svelte';
  import Note from '../services/Note';

  export let song, time, playing, mic;

  $: {
    const line = song.find(line => line.end > time && time > line.start);

    if (line)
    {
      const currentSyllable = line.syllables.find(syllable => syllable.start < time && time < syllable.start + syllable.length);

      if (currentSyllable)
      {
        mic.sung = checkMic(mic.sung, currentSyllable, mic.getPitch());
        mic.score = getScore(mic.sung);
      }
    }
  };

  function checkMic(sungData, syllable, input)
  {
    if (input !== null)
    {
      const sung = sungData[syllable.start] = sungData[syllable.start] || [];
      const lastSung = sung[sung.length - 1];
      const start = (time - syllable.start) * (100 / syllable.length);
      const match = getMatchingClass(syllable, input);
      const points = getMatchingPoints(syllable, input);

      if (lastSung)
      {
        lastSung.end = 100 - start;
      }
      if (!lastSung || lastSung.match !== match)
      {
        sung.push({ start, match, points });
      }
    }

    return sungData;
  }

  function getMatchingPoints(syllable, input)
  {
    // match golden note            = p * 2.5
    // match regular                = p * 1.5
    // match freestyle              = p * 1
    // too low/high / unknown pitch = p * 0.5

    if (syllable.type === 0)
    {
      return 1;
    }

    if (input.note)
    {
      const note = new Note(syllable.pitch);
      const diff = input.note - note;

      if (Math.abs(diff) < 2) // allow half-note tolerance
      {
        return syllable.type + 0.5;
      }
    }

    return 0.5;
  }

  function getMatchingClass(syllable, input)
  {
    if (syllable.type === 0)
    {
      return 'match';
    }

    if (input.note)
    {
      const note = new Note(syllable.pitch);
      const diff = input.note - note;

      if (Math.abs(diff) < 2) // allow half-note tolerance
      {
        return 'match';
      }
      else if (diff < 0)
      {
        return 'too-low';
      }
      else
      {
        return 'too-high';
      }
    }
  }

  function getScore(sungData)
  {
    return parseInt(Object
      .values(sungData)
      .reduce((sumA, syllable) =>
        syllable.reduce((sumB, item) =>
          sumB + ((100 - item.end - item.start) * item.points || 0), sumA), 0), 10);
  }

  function notePosition({ attributeStyleMap }, { line, syllable })
  {
    const factor = 100 / (line.end - line.start);

    attributeStyleMap.set('top', CSS.percent(((syllable.pitch - line.minPitch || 1) * 90) / (line.maxPitch - line.minPitch || 2)));
    attributeStyleMap.set('left', CSS.percent((syllable.start - line.start) * factor));
    attributeStyleMap.set('width', CSS.percent(syllable.length * factor));
  }
</script>

<style>
  .notes
  {
    position: relative;
    height: 100%;
    margin: 0 15%;
    flex: 1;
  }

  :global(.notes.red)
  {
    color: red;
  }

  :global(.notes.blue)
  {
    color: deepskyblue;
  }

  .note
  {
    position: absolute;
    height: 20px;
  }

  .note.golden
  {
    box-shadow: 0 0 15px 5px goldenrod;
    border-radius: 6px;
  }

  .note > div
  {
    position: absolute;
    width: 100%;
    height: 100%;
    border-radius: 6px;
    border: 1px solid transparent;
  }

  .note .pill
  {
    border: 1px solid rgba(255,255,255, .8);
    background: linear-gradient(
      hsla(0, 0%, 100%, 0.6),
      hsla(0, 0%, 100%, 0) 50%,
      hsla(0, 0%, 0%, 0.3) 50%,
      hsla(0, 0%, 100%, 0.2)
    );
    box-shadow: inset 0 -5px 20px hsla(0, 0%, 0%, 0.4),
                inset 0 5px 20px hsla(0, 0%, 100%, 0.4),
                -8px 8px 5px hsla(0, 0%, 0%, 0.15),
                5px 18px 10px hsla(0, 0%, 0%, 0.2);
  }

  .note .time
  {
    background-color: #fff;
    width: 0;
    opacity: 0;
    animation: grow var(--duration) linear var(--offset) forwards;
    animation-play-state: inherit;
  }

  .note .sung .running
  {
    will-change: right;
  }

  .note .sung .match
  {
    position: absolute;
    top: 0;
    bottom: 0;
    background: currentColor;
  }

  .note .sung .too-high,
  .note .sung .too-low
  {
    position: absolute;
    height: 10px;
    background: currentColor;
    border-radius: 3px;
    z-index: 1;
    opacity: .5;
    margin: 1px 0;
  }

  .note .sung .too-low
  {
    top: 100%;
  }

  .note .sung .too-high
  {
    bottom: 100%;
  }

  .inactive
  {
    animation-play-state: paused;
  }

  @keyframes grow
  {
    1% { opacity: 1; }
    to { opacity: 1; width: 100%; } /* TODO animating width is bad for performence */
  }

  .score
  {
    position: absolute;
    top: 50%;
    left: 100%;
    width: 15%;
    transform: translateY(-50%);
    font-size: 40px;
    color: #fff;
    text-shadow: 0 0 7px #000;
  }
</style>

<div class="notes {mic.color}" class:inactive={!playing}>
  {#each song as line}
    {#if line.end > time && time > line.start}
      {#each line.syllables as syllable}
        {#if syllable.type}
          <div class="note" use:notePosition={{line, syllable}} class:golden={syllable.type === 2}>
            <div class="time" style="--duration: {syllable.length}ms; --offset: {syllable.start - line.start}ms"></div>
            <div class="sung">
              {#if mic.sung[syllable.start]}
                {#each mic.sung[syllable.start] as sung}
                  {#if sung.end && sung.match}
                    <div class={sung.match}
                         class:running={syllable.start + syllable.length > time}
                         style="left: {sung.start}%; right: {sung.end}%">
                    </div>
                  {/if}
                {/each}
              {/if}
            </div>
            <div class="pill"></div>
          </div>
        {/if}
      {/each}
    {/if}
  {/each}
  {#if mic.score}
    <div class="score">{mic.score}</div>
  {/if}
</div>