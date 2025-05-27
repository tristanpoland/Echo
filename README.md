## Getting Started with Audio Context

The foundation of any web audio application is the AudioContext. Think of it as your audio processing hub - everything flows through it. However, there's an important gotcha that trips up many developers: browsers require a user interaction before you can create or resume an audio context. This is a security feature to prevent websites from immediately blasting sound at unsuspecting users.

```javascript
class AudioCaptureManager {
    constructor() {
        this.audioContext = null;
        this.micStream = null;
        this.systemStream = null;
        this.isInitialized = false;
        
        this.bindEvents();
    }
    
    async initializeAudio() {
        try {
            // This MUST happen after a user click/tap
            this.audioContext = new (window.AudioContext || window.webkitAudioContext)();
            
            if (this.audioContext.state === 'suspended') {
                await this.audioContext.resume();
            }
            
            // Request mic permission to enable device enumeration
            const tempStream = await navigator.mediaDevices.getUserMedia({ 
                audio: { echoCancellation: false, noiseSuppression: false } 
            });
            tempStream.getTracks().forEach(track => track.stop());
            
            await this.loadMicrophones();
            this.isInitialized = true;
            
        } catch (error) {
            console.error('Audio initialization failed:', error);
        }
    }
}
```

The temporary stream creation might seem wasteful, but it's actually essential. Browser security prevents you from enumerating audio devices until you've been granted microphone permission. We create a temporary stream just to trigger the permission dialog, then immediately close it so we can see what devices are available.

## Microphone Enumeration and Selection

One of the most frustrating aspects of web audio development is dealing with microphone device enumeration. Sometimes you get proper device names, sometimes you get "Microphone 1", and sometimes the list is completely empty even though you know there are microphones connected. The key is building a robust system that handles all these edge cases.

```javascript
async loadMicrophones() {
    const devices = await navigator.mediaDevices.enumerateDevices();
    const audioInputs = devices.filter(device => device.kind === 'audioinput');
    
    const micSelect = document.getElementById('micSelect');
    micSelect.innerHTML = '<option value="">Select microphone...</option>';
    
    // Always add a default option - this often works when specific devices don't
    const defaultOption = document.createElement('option');
    defaultOption.value = 'default';
    defaultOption.textContent = 'Default Microphone';
    micSelect.appendChild(defaultOption);
    
    audioInputs.forEach((device, index) => {
        const option = document.createElement('option');
        option.value = device.deviceId;
        // Fallback naming when labels aren't available
        option.textContent = device.label || `Microphone ${index + 1}`;
        micSelect.appendChild(option);
    });
}
```

The "Default Microphone" option is your safety net. Even when device enumeration fails or returns cryptic device IDs, the browser's default device selection usually works. I learned this the hard way after users complained that my app couldn't find their microphones even though other apps worked fine.

## Capturing and Routing Microphone Audio

Once you have device selection working, the actual audio capture is relatively straightforward. The Web Audio API uses a node-based system where you connect audio sources to processors to destinations. For a simple microphone echo effect, you're essentially creating a path from your microphone to your speakers with some processing in between.

```javascript
async startMicrophone() {
    const deviceId = document.getElementById('micSelect').value;
    if (!deviceId) throw new Error('Please select a microphone');
    
    // Use 'ideal' instead of 'exact' for better compatibility
    const constraints = deviceId === 'default' ? 
        { audio: { echoCancellation: false, noiseSuppression: false } } :
        { audio: { deviceId: { ideal: deviceId }, echoCancellation: false, noiseSuppression: false } };
    
    this.micStream = await navigator.mediaDevices.getUserMedia(constraints);
    
    // Create the audio processing chain
    this.micSource = this.audioContext.createMediaStreamSource(this.micStream);
    this.micGainNode = this.audioContext.createGain();
    this.micAnalyzer = this.audioContext.createAnalyser();
    
    // Configure the analyzer for visualization
    this.micAnalyzer.fftSize = 512;
    this.micAnalyzer.smoothingTimeConstant = 0.3;
    
    // Audio routing: mic -> volume control -> speakers AND analyzer
    this.micSource.connect(this.micGainNode);
    this.micGainNode.connect(this.audioContext.destination); // This creates the echo
    this.micGainNode.connect(this.micAnalyzer); // This enables visualization
    
    this.startVolumeVisualization();
}
```

The key insight here is that you can connect one audio node to multiple destinations. The microphone feeds into a gain node (for volume control), which then splits to both the speakers (for the echo effect) and an analyzer node (for visual feedback). This pattern of splitting audio streams is fundamental to building complex audio applications.

## System Audio Capture - The Screen Sharing Workaround

System audio capture is where things get interesting - and by interesting, I mean incredibly frustrating. Due to browser security restrictions, there's no direct way to capture system audio. The only workaround is to use screen sharing with audio enabled. Yes, you read that right - you have to share your screen to capture audio, even if you don't care about the video.

```javascript
async startSystemAudio() {
    try {
        // Request screen share with audio - this is the only way to get system audio
        this.systemStream = await navigator.mediaDevices.getDisplayMedia({
            video: { mediaSource: 'screen' }, // Required even though we only want audio
            audio: { echoCancellation: false, noiseSuppression: false }
        });
        
        // Verify audio track exists - user must manually enable "Share system audio"
        const audioTracks = this.systemStream.getAudioTracks();
        if (audioTracks.length === 0) {
            throw new Error('No system audio captured. Make sure to check "Share system audio" in the dialog.');
        }
        
        // Same audio routing pattern as microphone
        this.systemSource = this.audioContext.createMediaStreamSource(this.systemStream);
        this.systemGainNode = this.audioContext.createGain();
        this.systemAnalyzer = this.audioContext.createAnalyser();
        
        this.systemSource.connect(this.systemGainNode);
        this.systemGainNode.connect(this.audioContext.destination);
        this.systemGainNode.connect(this.systemAnalyzer);
        
        // Handle user stopping screen share
        this.systemStream.getVideoTracks()[0].addEventListener('ended', () => {
            this.stopSystemAudio();
        });
        
    } catch (error) {
        console.error('System audio capture failed:', error);
    }
}
```

This limitation isn't a bug - it's a deliberate security feature. Browsers don't want malicious websites secretly recording your system audio, so they require explicit user consent through the screen sharing dialog. The user must manually check the "Share system audio" option, and there's no way to programmatically force this. Make sure your UI clearly explains this requirement to users.

## Real-Time Audio Visualization

Visual feedback is crucial for audio applications. Users need to see that audio is being captured and processed. The Web Audio API provides analyzer nodes that can extract frequency data for visualization. The implementation is straightforward but requires understanding how to work with frequency domain data.

```javascript
startVolumeVisualization() {
    if (this.animationId) return; // Prevent multiple animation loops
    
    const dataArray = new Uint8Array(256);
    
    const updateBars = () => {
        // Process microphone visualization
        if (this.micAnalyzer) {
            this.micAnalyzer.getByteFrequencyData(dataArray);
            const average = dataArray.reduce((sum, value) => sum + value, 0) / dataArray.length;
            const percentage = Math.min((average / 128) * 100, 100);
            document.getElementById('micVolumeBar').style.width = percentage + '%';
        }
        
        // Process system audio visualization
        if (this.systemAnalyzer) {
            this.systemAnalyzer.getByteFrequencyData(dataArray);
            const average = dataArray.reduce((sum, value) => sum + value, 0) / dataArray.length;
            const percentage = Math.min((average / 128) * 100, 100);
            document.getElementById('systemVolumeBar').style.width = percentage + '%';
        }
        
        this.animationId = requestAnimationFrame(updateBars);
    };
    
    updateBars();
}
```

The analyzer gives you frequency data as an array of values from 0-255. For a simple volume meter, we average all the frequency bins to get an overall amplitude. For more sophisticated visualizations, you could create spectrum analyzers by mapping different frequency ranges to different visual elements.

## Volume Controls and Audio Processing

Adding volume controls involves connecting gain nodes in your audio chain. Gain nodes are incredibly useful - they can amplify, attenuate, or even completely mute audio streams. The key is connecting them properly in your audio graph and updating their values in response to user input.

```javascript
setupVolumeControls() {
    document.getElementById('micVolume').addEventListener('input', (e) => {
        const value = e.target.value / 100; // Convert 0-100 to 0-1
        if (this.micGainNode) {
            this.micGainNode.gain.value = value;
        }
    });
    
    document.getElementById('systemVolume').addEventListener('input', (e) => {
        const value = e.target.value / 100;
        if (this.systemGainNode) {
            this.systemGainNode.gain.value = value;
        }
    });
}
```

Gain values in the Web Audio API use a linear scale where 1.0 is unity gain (no change), values less than 1.0 reduce volume, and values greater than 1.0 amplify the signal. Be careful with amplification as it can cause clipping and distortion.

## Resource Management and Cleanup

One aspect that's often overlooked in tutorials is proper resource cleanup. Audio streams, analyzer nodes, and gain nodes all consume system resources. If you don't clean them up properly, you'll create memory leaks that can crash browsers or degrade performance over time.

```javascript
stopMicrophone() {
    // Stop all media tracks
    if (this.micStream) {
        this.micStream.getTracks().forEach(track => track.stop());
        this.micStream = null;
    }
    
    // Disconnect all audio nodes
    if (this.micSource) {
        this.micSource.disconnect();
        this.micSource = null;
    }
    
    if (this.micGainNode) {
        this.micGainNode.disconnect();
        this.micGainNode = null;
    }
    
    if (this.micAnalyzer) {
        this.micAnalyzer.disconnect();
        this.micAnalyzer = null;
    }
    
    this.checkStopVisualization();
}

checkStopVisualization() {
    // Only stop animation loop when no analyzers are active
    if (!this.micAnalyzer && !this.systemAnalyzer && this.animationId) {
        cancelAnimationFrame(this.animationId);
        this.animationId = null;
    }
}
```

The order matters here. Always stop media tracks first, then disconnect audio nodes, and finally clean up any animation loops. Setting references to null helps the garbage collector reclaim memory more aggressively.

## Browser Compatibility and Edge Cases

Different browsers handle audio APIs differently, and you'll encounter various edge cases in production. Chrome generally has the best Web Audio API support, Firefox is solid but sometimes has different default behaviors, and Safari works well on newer versions but has some quirks with older implementations. Edge (Chromium-based) is generally compatible with Chrome's behavior.

The most common issues I've encountered include audio contexts getting suspended unexpectedly (especially on mobile), device enumeration returning empty lists, and permission dialogs behaving differently across browsers. Always include fallbacks and graceful error handling.