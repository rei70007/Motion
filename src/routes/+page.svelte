<script>
    import { onMount } from 'svelte';
    

    let alpha = $state(0);
    let beta = $state(0);
    let gamma = $state(0);

    onMount(() => {
        if (window.DeviceOrientationEvent) {
            window.addEventListener('deviceorientation', (event) => {
                alpha = event.alpha;
                beta = event.beta;
                gamma = event.gamma;
            });
        } else {
            console.log('Device orientation not supported');
        }
    });
</script>

<h1>Device Orientation</h1>

<p>Alpha: {alpha}</p>
<p>Beta: {beta}</p>
<p>Gamma: {gamma}</p>