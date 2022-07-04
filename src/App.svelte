<script>
  const OBS_WEBSOCKET_LATEST_VERSION = '4.9.1'; // https://api.github.com/repos/Palakis/obs-websocket/releases/latest

  // Imports
  import { onMount } from 'svelte';
  import { mdiSquareRoundedBadge, mdiSquareRoundedBadgeOutline, mdiImageEdit, mdiImageEditOutline, mdiFullscreen, mdiFullscreenExit, mdiBorderVertical, mdiArrowSplitHorizontal, mdiAccessPoint, mdiAccessPointOff, mdiRecord, mdiStop, mdiPause, mdiPlayPause, mdiConnection } from '@mdi/js';
  import Icon from 'mdi-svelte';
  import compareVersions from 'compare-versions';

  import './style.scss';
  import { obs, sendCommand } from './obs.js';
  import ProgramPreview from './ProgramPreview.svelte';
  import SceneSwitcher from './SceneSwitcher.svelte';
  import SourceSwitcher from './SourceSwitcher.svelte';
  import ProfileSelect from './ProfileSelect.svelte';
  import SceneCollectionSelect from './SceneCollectionSelect.svelte';

  // Constants
  const obs_host = process.env.OBS_HOST;

  onMount(async () => {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/service-worker.js');
    }

    // Request screen wakelock
    if ('wakeLock' in navigator) {
      try {
        wakeLock = await navigator.wakeLock.request('screen');
          // Re-request when coming back
          document.addEventListener('visibilitychange', async () => {
            if (document.visibilityState === 'visible') {
              wakeLock = await navigator.wakeLock.request('screen');
            }
          });
      }
      catch(e) { }
    }

    // Toggle the navigation hamburger menu on mobile
    const navbar = document.querySelector('.navbar-burger');
    navbar.addEventListener('click', () => {
      navbar.classList.toggle('is-active');
      document.getElementById(navbar.dataset.target).classList.toggle('is-active');
    });

    // Listen for fullscreen changes
    document.addEventListener('fullscreenchange', () => {
      isFullScreen = document.fullscreenElement;
    });

    document.addEventListener('webkitfullscreenchange', () => {
      isFullScreen = document.webkitFullscreenElement;
    });

    document.addEventListener('msfullscreenchange', () => {
      isFullScreen = document.msFullscreenElement;
    });

    if (document.location.hash !== '') {
      // Read address from hash
      address = document.location.hash.slice(1);
      await connect();
    }
  });

  // State
  let connected,
    heartbeat = false,
    isFullScreen,
    isStudioMode,
    isSceneOnTop,
    isIconMode = window.localStorage.getItem('isIconMode') || false,
    editable = false,
    wakeLock = false,
    address,
    password,
    scenes = [],
    errorMessage = '';

  $: isIconMode 
      ? window.localStorage.setItem('isIconMode', "true")
      : window.localStorage.removeItem('isIconMode');
  
  function formatTime(secs) {
    let hours = Math.floor(secs / 3600);
    secs -= hours * 3600;
    let mins = Math.floor(secs / 60);
    secs -= mins * 60;
    return (hours > 0)
      ? `${hours}:${mins<10?'0':''}${mins}:${secs<10?'0':''}${secs}`
      : `${mins<10?'0':''}${mins}:${secs<10?'0':''}${secs}`;
  }

  function toggleFullScreen() {
    if (isFullScreen) {
      if (document.exitFullscreen) {
        document.exitFullscreen();
      } else if (document.webkitExitFullscreen) {
        document.webkitExitFullscreen();
      } else if (document.msExitFullscreen) {
        document.msExitFullscreen();
      }
    } else {
      if (document.documentElement.requestFullscreen) {
        document.documentElement.requestFullscreen();
      } else if (document.documentElement.webkitRequestFullscreen) {
        document.documentElement.webkitRequestFullscreen();
      } else if (document.documentElement.msRequestFullscreen) {
        document.documentElement.msRequestFullscreen();
      }
    }
  }

  async function toggleStudioMode() {
    await sendCommand('ToggleStudioMode');
  }

  async function switchSceneView() {
    isSceneOnTop = !isSceneOnTop;
  }

  async function startStream() {
    await sendCommand('StartStreaming');
  }

  async function stopStream() {
    await sendCommand('StopStreaming');
  }

  async function startRecording() {
    await sendCommand('StartRecording');
  }

  async function stopRecording() {
    await sendCommand('StopRecording');
  }

  async function pauseRecording(){
    await sendCommand('PauseRecording');
  }

  async function resumeRecording(){
    await sendCommand('ResumeRecording');
  }

  async function connect() {
    address = obs_host;
    let secure = location.protocol === 'https:' || address.endsWith(':443');
    if (address.indexOf('://') !== -1) {
      let url = new URL(address);
      secure = url.protocol === 'wss:' || url.protocol === 'https:';
      address = url.hostname + ':' + (url.port ? url.port : secure ? 443 : 80);
    }
    console.log('Connecting to:', address, '- secure:', secure, '- using password:', password);
    await disconnect();
    try {
      await obs.connect({ address: address, password, secure });
    } catch (e) {
      console.log(e);
      errorMessage = e.description;
    }
  }

  async function disconnect() {
    await obs.disconnect();
    connected = false;
    errorMessage = 'Déconnecté';
  }

  async function onKeyup(event) {
    if (event.key === 'Enter') {
      await connect();
      event.preventDefault();
    }
  }

  // OBS events
  obs.on('ConnectionClosed', () => {
    connected = false;
    window.history.pushState('', document.title, window.location.pathname + window.location.search); // Remove the hash
    console.log('Connexion rompue');
  });

  obs.on('AuthenticationSuccess', async () => {
    console.log('Connexion établie');
    connected = true;
    document.location.hash = address; // For easy bookmarking
    const version = (await sendCommand('GetVersion')).obsWebsocketVersion || '';
    console.log('OBS-websocket version:', version);
    if(compareVersions(version, OBS_WEBSOCKET_LATEST_VERSION) < 0) {
      alert('Visiblement nous utilisons une ancienne version de OBS-websocket (version ' + version + '), il serait temps de contacter Michel.');
    }
    await sendCommand('SetHeartbeat', { enable: true });
    let data = await sendCommand('GetStudioModeStatus');
    isStudioMode = (data && data.studioMode) || false;
  });

  obs.on('AuthenticationFailure', async () => {
    errorMessage = 'Please enter your password:';
    document.getElementById('password').focus();
    if (!password) {
      connected = false;
    } else {
      await connect();
    }
  });

  // Heartbeat
  obs.on('Heartbeat', data => {
    heartbeat = data;
  });

  obs.on('StudioModeSwitched', async (data) => {
    console.log('StudioModeSwitched', data.newState);
    isStudioMode = (data && data.studioMode) || false;
  });
</script>

<svelte:head>
  <title>AV Switcher ─ Dreamsee Télévision</title>
</svelte:head>

<nav class="navbar is-primary" aria-label="main navigation">
  <div class="navbar-brand">
    <a class="navbar-item is-size-4 has-text-weight-bold" href="/">
      <img src="favicon.png" alt="AV Switcher" class="rotate" />
      <img height="25px" src="https://cdn.dreamsee.tv/img/logo/dreamsee.png" alt="dreamsee.tv" />
    </a>

    <!-- svelte-ignore a11y-missing-attribute -->
    <button class="navbar-burger burger" aria-label="menu" aria-expanded="false" data-target="navmenu">
      <span aria-hidden="true" />
      <span aria-hidden="true" />
      <span aria-hidden="true" />
    </button>
  </div>

  <div id="navmenu" class="navbar-menu">
    <div class="navbar-end">
      <div class="navbar-item">
        <div class="buttons">
          <!-- svelte-ignore a11y-missing-attribute -->
          {#if connected}
            <button class="button is-info is-light" disabled>
              {#if heartbeat}
                {Math.round(heartbeat.stats.fps)} fps, {Math.round(heartbeat.stats['cpu-usage'])}% CPU, {heartbeat.stats['output-skipped-frames']} images perdues
              {:else}Connecté{/if}
            </button>
            {#if heartbeat && heartbeat.streaming}
              <button class="button is-danger" on:click={stopStream} title="ARRÊT DIFF">
                <span class="icon"><Icon path={mdiAccessPointOff} /></span>
                <span>{formatTime(heartbeat.totalStreamTime)}</span>
              </button>
            {:else}
              <button class="button is-danger is-light" on:click={startStream} title="DÉMARRER DIFF">
                <span class="icon"><Icon path={mdiAccessPoint} /></span>
              </button>
            {/if}
            {#if heartbeat && heartbeat.recording}
              {#if heartbeat.recordingPaused}
                <button class="button is-danger" on:click={resumeRecording} title="Reprendre enregistrement">
                  <span class="icon"><Icon path={mdiPlayPause} /></span>
                </button>
              {:else}
                <button class="button is-success" on:click={pauseRecording} title="Pause enregistrement">
                  <span class="icon"><Icon path={mdiPause} /></span>
                </button>
              {/if}
              <button class="button is-danger" on:click={stopRecording} title="Arrêt enregistrement">
                <span class="icon"><Icon path={mdiStop} /></span>
                <span>{formatTime(heartbeat.totalRecordTime)}</span>
              </button>
            {:else}
              <button class="button is-danger is-light" on:click={startRecording} title="Démarrer enregistrement">
                <span class="icon"><Icon path={mdiRecord} /></span>
              </button>
            {/if}
            <button class:is-light={!isStudioMode} class="button is-link" on:click={toggleStudioMode} title="Activer Mode Studio">
              <span class="icon"><Icon path={mdiBorderVertical} /></span>
            </button>
            <button class:is-light={!isSceneOnTop} class="button is-link" on:click={switchSceneView} title="Voir scène en haut">
              <span class="icon"><Icon path={mdiArrowSplitHorizontal} /></span>
            </button>
            <button class:is-light={!editable} class="button is-link" title="Modifier scènes" on:click={()=>(editable=!editable)}>
              <span class="icon">
                <Icon path={editable ? mdiImageEditOutline : mdiImageEdit} />
              </span>
            </button>
            <button class:is-light={!isIconMode} class="button is-link" title="Voir scènes en icônes" on:click={()=>(isIconMode=!isIconMode)}>
              <span class="icon">
                <Icon path={isIconMode ? mdiSquareRoundedBadgeOutline : mdiSquareRoundedBadge} />
              </span>
            </button>
            <ProfileSelect />
            <SceneCollectionSelect />
            <button class="button is-danger is-light" on:click={disconnect} title="Se déconnecter">
              <span class="icon"><Icon path={mdiConnection} /></span>
            </button>
          {:else}
            <button class="button is-danger" disabled>{errorMessage || 'Déconnecté'}</button>
          {/if}
          <!-- svelte-ignore a11y-missing-attribute -->
          <button class:is-light={!isFullScreen} class="button is-link" on:click={toggleFullScreen} title="Activer Mode Plein Écran">
            <span class="icon">
              <Icon path={isFullScreen ? mdiFullscreenExit : mdiFullscreen} />
            </span>
          </button>
        </div>
      </div>
    </div>
  </div>
</nav>

<section class="section">
  <div class="container">
    {#if connected}
      {#if isSceneOnTop}
        <ProgramPreview />
      {/if}
      <SceneSwitcher bind:scenes={scenes} buttonStyle={isIconMode ? "icon" : "text"} editable={editable} />
      {#if !isSceneOnTop}
        <ProgramPreview />
      {/if}
      {#each scenes as scene}
        {#if scene.name.indexOf('(switch)') > 0}
        <SourceSwitcher name={scene.name} buttonStyle="screenshot" />
        {/if}
      {/each}
    {:else}
      <h1 class="subtitle">
        Bienvenue sur 
        <strong>AV Switcher</strong>
        - Panneau de gestion de l'encodeur de la Dreamsee Télévision
      </h1>
    
      <div class="notification is-danger">
          L'outil est réservé <strong>uniquement</strong> aux personnes ayant l'autorisation de l'équipe technique.
		      <br>Cette version de AV Switcher est en développement, des améliorations et mises à jours sont prévus.
          <br><br>
          <strong><a href="mailto:contact+avmystery@dreamsee.tv">
          Si tu as rien à faire ici (ou que tu as trouvé "mystérieusement" cette porte) je penses que tu devrais travailler avec nous
          </a></strong> !
      </div>

      <p>Pour démarrer la session, inscris le mot de passe ci-dessous et appuies sur le bouton "Se connecter".</p>

      <div class="field is-grouped">
        <p class="control is-expanded">
          <input id="host" on:keyup={onKeyup} bind:value={address} class="input" type="text" placeholder="localhost:4444" />
          <input id="password" on:keyup={onKeyup} bind:value={password} class="input" type="password" placeholder="Mot de passe" />
        </p>
        <p class="control">
          <button on:click={connect} class="button is-success">Se connecter</button>
        </p>

      </div>
    {/if}
  </div>

</section>

<footer class="footer">
  <div class="content has-text-centered">
    <p style="color:black">
      <strong>AV Switcher, un fork de OBS-web</strong>
      conçu originalement par <a href="https://niekvandermaas.nl/">Niek van der Maas</a>
      &mdash; Le repository 
      <a href="https://github.com/Niek/obs-web">GitHub</a>
      originale est disponible.
    </p>
  </div>
</footer>
