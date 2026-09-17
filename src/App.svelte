<script lang="ts">
  import { onMount } from 'svelte'

  type Team = {
    id: number
    name: string
    avatar: string
    color: string
    score: number
  }

  type Guess = {
    value: number
    teamName: string
    avatar: string
    direction: 'higher' | 'lower' | 'correct'
  }

  const avatars = ['🦊', '🐸', '🐼', '🐯', '🐙', '🦄', '🦁', '🐧']
  const colors = ['coral', 'blue', 'yellow', 'purple', 'green', 'pink']
  const presets = [
    { label: '1 – 20', min: 1, max: 20 },
    { label: '1 – 100', min: 1, max: 100 },
    { label: '1 – 1,000', min: 1, max: 1000 },
    { label: '1 – 10,000', min: 1, max: 10000 },
    { label: '−100 – 100', min: -100, max: 100 },
  ]

  let screen = $state<'setup' | 'game'>('setup')
  let rangeMin = $state(1)
  let rangeMax = $state(100)
  let selectedPreset = $state('1 – 100')
  let teams = $state<Team[]>([
    { id: 1, name: 'Clever Foxes', avatar: '🦊', color: 'coral', score: 0 },
    { id: 2, name: 'Brilliant Frogs', avatar: '🐸', color: 'blue', score: 0 },
  ])
  let nextTeamId = 3
  let target = $state(0)
  let currentMin = $state(1)
  let currentMax = $state(100)
  let activeTeam = $state(0)
  let roundStarter = $state(0)
  let guessInput = $state('')
  let guesses = $state<Guess[]>([])
  let winnerIndex = $state<number | null>(null)
  let errorMessage = $state('')
  let round = $state(1)

  const active = $derived(teams[activeTeam])
  const remaining = $derived(Math.max(0, currentMax - currentMin + 1))
  const totalNumbers = $derived(Math.max(1, rangeMax - rangeMin + 1))
  const leftPercent = $derived(((currentMin - rangeMin) / totalNumbers) * 100)
  const rightPercent = $derived(((rangeMax - currentMax) / totalNumbers) * 100)
  const latestGuess = $derived(guesses[0])

  onMount(() => {
    const saved = localStorage.getItem('number-showdown-teams')
    if (saved) {
      try {
        const parsed = JSON.parse(saved)
        if (Array.isArray(parsed) && parsed.length >= 2) {
          teams = parsed
          nextTeamId = Math.max(...teams.map((team) => team.id)) + 1
        }
      } catch {
        // Start fresh when saved data is malformed.
      }
    }

    const beaconToken = import.meta.env.VITE_CF_BEACON
    if (beaconToken) {
      const script = document.createElement('script')
      script.defer = true
      script.src = 'https://static.cloudflareinsights.com/beacon.min.js'
      script.dataset.cfBeacon = JSON.stringify({ token: beaconToken })
      document.head.appendChild(script)
    }
  })

  function saveTeams() {
    localStorage.setItem('number-showdown-teams', JSON.stringify(teams))
  }

  function choosePreset(preset: (typeof presets)[number]) {
    rangeMin = preset.min
    rangeMax = preset.max
    selectedPreset = preset.label
  }

  function updateCustomRange() {
    selectedPreset = 'custom'
  }

  function addTeam() {
    if (teams.length >= 6) return
    const index = teams.length
    teams.push({
      id: nextTeamId++,
      name: `Team ${index + 1}`,
      avatar: avatars[index % avatars.length],
      color: colors[index % colors.length],
      score: 0,
    })
  }

  function removeTeam(index: number) {
    if (teams.length <= 2) return
    teams.splice(index, 1)
  }

  function cycleAvatar(index: number) {
    const avatarIndex = avatars.indexOf(teams[index].avatar)
    teams[index].avatar = avatars[(avatarIndex + 1) % avatars.length]
  }

  function randomTarget() {
    return Math.floor(Math.random() * (rangeMax - rangeMin + 1)) + rangeMin
  }

  function startGame() {
    errorMessage = ''
    if (!Number.isInteger(rangeMin) || !Number.isInteger(rangeMax)) {
      errorMessage = 'Please use whole numbers for the range.'
      return
    }
    if (rangeMin >= rangeMax) {
      errorMessage = 'The highest number needs to be greater than the lowest.'
      return
    }
    if (rangeMax - rangeMin > 1_000_000_000) {
      errorMessage = 'Please choose a range of one billion numbers or fewer.'
      return
    }
    teams.forEach((team, index) => {
      if (!team.name.trim()) team.name = `Team ${index + 1}`
    })
    saveTeams()
    round = 1
    roundStarter = 0
    activeTeam = 0
    beginRound()
    screen = 'game'
  }

  function beginRound() {
    target = randomTarget()
    currentMin = rangeMin
    currentMax = rangeMax
    guesses = []
    winnerIndex = null
    guessInput = ''
    errorMessage = ''
    activeTeam = roundStarter % teams.length
    requestAnimationFrame(() => document.querySelector<HTMLInputElement>('#guess')?.focus())
  }

  function submitGuess(event: SubmitEvent) {
    event.preventDefault()
    if (winnerIndex !== null) return

    const value = Number(guessInput)
    if (!guessInput.trim() || !Number.isInteger(value)) {
      errorMessage = 'Enter a whole number to make a guess.'
      return
    }
    if (value < currentMin || value > currentMax) {
      errorMessage = `Your guess must be between ${formatNumber(currentMin)} and ${formatNumber(currentMax)}.`
      return
    }

    errorMessage = ''
    const guessingTeam = teams[activeTeam]
    if (value === target) {
      guesses.unshift({ value, teamName: guessingTeam.name, avatar: guessingTeam.avatar, direction: 'correct' })
      winnerIndex = activeTeam
      teams[activeTeam].score += 1
      saveTeams()
      return
    }

    const direction = value < target ? 'higher' : 'lower'
    guesses.unshift({ value, teamName: guessingTeam.name, avatar: guessingTeam.avatar, direction })
    if (direction === 'higher') currentMin = value + 1
    else currentMax = value - 1

    activeTeam = (activeTeam + 1) % teams.length
    guessInput = ''
    requestAnimationFrame(() => document.querySelector<HTMLInputElement>('#guess')?.focus())
  }

  function nextRound() {
    round += 1
    roundStarter = (roundStarter + 1) % teams.length
    beginRound()
  }

  function editGame() {
    screen = 'setup'
    winnerIndex = null
    errorMessage = ''
  }

  function resetScores() {
    teams.forEach((team) => (team.score = 0))
    saveTeams()
  }

  function formatNumber(value: number) {
    return new Intl.NumberFormat('en-US').format(value)
  }
</script>

<svelte:head>
  <title>Number Showdown — A classroom guessing game</title>
  <meta name="description" content="A fast, team-based higher or lower number guessing game for the classroom." />
</svelte:head>

{#if screen === 'setup'}
  <main class="setup-page">
    <header class="topbar">
      <a class="brand" href="/" aria-label="Number Showdown home">
        <span class="brand-mark" aria-hidden="true">#</span>
        <span>Number Showdown</span>
      </a>
      <span class="tagline">A classroom guessing game</span>
    </header>

    <section class="setup-shell">
      <div class="setup-intro">
        <span class="eyebrow"><span></span> READY, SET, GUESS!</span>
        <h1>Set up your<br /><em>showdown.</em></h1>
        <p>Pick a secret-number range, rally your teams, and let the guessing begin.</p>
        <div class="mascot-line" aria-hidden="true">
          <span class="mini-mascot m1">🦊</span>
          <span class="mini-mascot m2">🐸</span>
          <span class="mini-mascot m3">🐼</span>
          <svg viewBox="0 0 180 38" preserveAspectRatio="none"><path d="M2 31 C48 4, 125 4, 178 29" /></svg>
        </div>
      </div>

      <div class="setup-card">
        <section class="setup-section">
          <div class="section-heading">
            <span class="step-number">1</span>
            <div><h2>Choose the range</h2><p>The secret number can be any whole number in this span.</p></div>
          </div>
          <div class="preset-grid">
            {#each presets as preset}
              <button class:active={selectedPreset === preset.label} onclick={() => choosePreset(preset)}>{preset.label}</button>
            {/each}
          </div>
          <div class="range-inputs">
            <label>Lowest number<input type="number" bind:value={rangeMin} oninput={updateCustomRange} /></label>
            <span class="range-arrow" aria-hidden="true">→</span>
            <label>Highest number<input type="number" bind:value={rangeMax} oninput={updateCustomRange} /></label>
          </div>
        </section>

        <div class="rule"></div>

        <section class="setup-section">
          <div class="section-heading team-heading">
            <span class="step-number blue">2</span>
            <div><h2>Name your teams</h2><p>Tap a mascot to swap it.</p></div>
            <span class="team-count">{teams.length} TEAMS</span>
          </div>
          <div class="team-editor-list">
            {#each teams as team, index (team.id)}
              <div class="team-editor-row">
                <button class="avatar-button {team.color}" onclick={() => cycleAvatar(index)} aria-label={`Change mascot for ${team.name}`}>{team.avatar}</button>
                <label>
                  <span>Team {index + 1}</span>
                  <input maxlength="28" bind:value={team.name} aria-label={`Team ${index + 1} name`} />
                </label>
                {#if teams.length > 2}<button class="remove-button" onclick={() => removeTeam(index)} aria-label={`Remove ${team.name}`}>×</button>{/if}
              </div>
            {/each}
          </div>
          {#if teams.length < 6}
            <button class="add-team" onclick={addTeam}><span>+</span> Add another team</button>
          {/if}
        </section>

        {#if errorMessage}<p class="form-error" role="alert">{errorMessage}</p>{/if}
        <button class="start-button" onclick={startGame}>Start the showdown <span>→</span></button>
      </div>
    </section>
    <footer>Made for big guesses, quick thinking, and a little friendly competition.</footer>
  </main>
{:else}
  <main class="game-page">
    <header class="game-header">
      <div class="brand compact"><span class="brand-mark">#</span><span>Number Showdown</span></div>
      <div class="round-label">ROUND <strong>{round}</strong></div>
      <button class="text-button" onclick={editGame}><span>⚙</span> Edit game</button>
    </header>

    <section class="scoreboard" aria-label="Scoreboard">
      {#each teams as team, index (team.id)}
        <div class="score-card {team.color}" class:active={activeTeam === index && winnerIndex === null} class:winner={winnerIndex === index}>
          <div class="score-avatar" class:celebrate={winnerIndex === index}>{team.avatar}</div>
          <div class="score-info"><span>{team.name}</span><strong>{team.score} <small>{team.score === 1 ? 'POINT' : 'POINTS'}</small></strong></div>
          {#if activeTeam === index && winnerIndex === null}<span class="turn-pill">YOUR TURN</span>{/if}
          {#if winnerIndex === index}<span class="turn-pill win-pill">WINNER!</span>{/if}
        </div>
      {/each}
    </section>

    <section class="game-card">
      {#if winnerIndex !== null}
        <div class="confetti" aria-hidden="true">
          {#each Array(18) as _, i}<i style={`--i:${i}`}></i>{/each}
        </div>
        <div class="win-state" aria-live="polite">
          <div class="winner-mascot">{teams[winnerIndex].avatar}<span>★</span></div>
          <p class="eyebrow centered">WE HAVE A WINNER!</p>
          <h1>{teams[winnerIndex].name}</h1>
          <p class="answer-copy">The secret number was <strong>{formatNumber(target)}</strong></p>
          <div class="win-actions">
            <button class="start-button" onclick={nextRound}>Play another round <span>→</span></button>
            <button class="secondary-button" onclick={editGame}>Change game setup</button>
          </div>
        </div>
      {:else}
        <div class="turn-prompt">
          <div class="current-avatar {active.color}">{active.avatar}</div>
          <div><span>IT'S YOUR TURN</span><h1>{active.name}, take a guess!</h1></div>
        </div>

        <div class="range-panel">
          <div class="range-meta">
            <span>THE SECRET NUMBER IS BETWEEN</span>
            <strong>{formatNumber(currentMin)} <small>and</small> {formatNumber(currentMax)}</strong>
          </div>
          <div class="number-track" aria-label={`Possible numbers from ${currentMin} to ${currentMax}`}>
            <div class="eliminated left" style={`width:${Math.min(100, Math.max(0, leftPercent))}%`}></div>
            <div class="possible" style={`left:${Math.min(100, Math.max(0, leftPercent))}%;right:${Math.min(100, Math.max(0, rightPercent))}%`}>
              <span class="pulse-dot"></span>
            </div>
            <div class="eliminated right" style={`width:${Math.min(100, Math.max(0, rightPercent))}%`}></div>
          </div>
          <div class="track-labels"><span>{formatNumber(rangeMin)}</span><span class="remaining">{formatNumber(remaining)} possible {remaining === 1 ? 'number' : 'numbers'}</span><span>{formatNumber(rangeMax)}</span></div>
        </div>

        {#if latestGuess}
          <div class="feedback {latestGuess.direction}" aria-live="polite">
            <span class="feedback-arrow">{latestGuess.direction === 'higher' ? '↑' : '↓'}</span>
            <div><strong>Go {latestGuess.direction}!</strong><small>{latestGuess.teamName} guessed {formatNumber(latestGuess.value)}</small></div>
          </div>
        {/if}

        <form class="guess-form" onsubmit={submitGuess}>
          <label for="guess">Enter a whole number</label>
          <div class="guess-row">
            <input id="guess" type="number" bind:value={guessInput} min={currentMin} max={currentMax} step="1" placeholder="Your guess" autocomplete="off" />
            <button type="submit">Lock in guess <span>→</span></button>
          </div>
          {#if errorMessage}<p class="form-error" role="alert">{errorMessage}</p>{/if}
        </form>
      {/if}
    </section>

    <section class="game-bottom">
      <div class="history">
        <h2>Guess history <span>{guesses.length}</span></h2>
        {#if guesses.length === 0}
          <p class="empty-history">No guesses yet. Who will make the first move?</p>
        {:else}
          <div class="history-list">
            {#each guesses.slice(0, 8) as guess}
              <div><span class="history-avatar">{guess.avatar}</span><span>{guess.teamName}</span><strong>{formatNumber(guess.value)}</strong><i class={guess.direction}>{guess.direction === 'higher' ? '↑ Higher' : guess.direction === 'lower' ? '↓ Lower' : '★ Correct'}</i></div>
            {/each}
          </div>
        {/if}
      </div>
      <div class="teacher-tools">
        <h2>Game controls</h2>
        <button onclick={resetScores}>↻ Reset scores</button>
        <button onclick={editGame}>⚙ Change teams or range</button>
      </div>
    </section>
  </main>
{/if}
