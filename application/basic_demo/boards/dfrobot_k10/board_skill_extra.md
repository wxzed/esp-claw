## Audio Usage Notes (DFRobot K10 specific)

The K10 board has two non-obvious wiring details that the `audio` Lua module
must be told about. **Always use the calls below verbatim**; do not invent
parameters such as `channels=4` for the microphone.

### Speaker (`audio_dac`, NS4168 amp)
- Mono speaker, wired to the I2S RIGHT slot.
- The Lua side passes `channels=1`; `lua_module_audio` internally opens the
  codec_dev as stereo and duplicates each mono sample to both slots, so the
  speaker plays no matter which slot the amp listens on.
- Sample rate: **24000 Hz**, 16-bit, mono.

```lua
local board = require("board_manager")
local audio = require("audio")
local dac_dev = board.get_device("audio_dac")
local dac = audio.new_output(dac_dev, 24000, 1, 16, 80)   -- volume 0..100
audio.play_wav(dac, "/fatfs/some_24k_mono.wav")           -- WAV must be 24kHz mono 16-bit
audio.close(dac)
```

### Microphone (`audio_adc`, ES7243E)
- Wired as **4-slot TDM** on I2S0 RX, sharing BCLK/WS with the speaker.
  - SLOT0 = mic, SLOT1 = AEC reference, SLOT2/3 = unused.
- The Lua side decides how many logical channels to expose, but **must always
  pass `hw_channels=4` as the 6th argument** to `audio.new_input` so the
  underlying I2S TDM controller is configured with the correct
  `total_slot=4`. Omitting it (or passing `channels=4` directly) produces
  garbled noise, because the codec wire layout (4 slots) and the I2S
  controller (defaulting to 2 slots) end up out of sync.
- Sample rate: **24000 Hz**, 16-bit. Recommended gain: `37.5` dB.

```lua
local mic_dev = board.get_device("audio_adc")

-- Record only the mic (mono): channels=1, hw_channels=4
local mic = audio.new_input(mic_dev, 24000, 1, 16, 37.5, 4)
audio.record_wav(mic, "/fatfs/rec.wav", 5000)            -- writes a 1-channel WAV
audio.close(mic)

-- Or record mic + AEC reference (stereo): channels=2, hw_channels=4
local mic2 = audio.new_input(mic_dev, 24000, 2, 16, 37.5, 4)
audio.record_wav(mic2, "/fatfs/rec2.wav", 5000)          -- writes a 2-channel WAV
audio.close(mic2)
```

### Combined notes
- Speaker and microphone share the same I2S0 BCLK; **always use 24 kHz on
  both sides** in the same script. Mixing 16 kHz playback with 24 kHz
  recording will reconfigure BCLK at runtime and corrupt the other direction.
- Always `audio.close(handle)` an input/output handle before opening a new
  one for the same direction.
- Wrong patterns to avoid:
  - `audio.new_input(mic_dev, 24000, 4, 16, 37.5)` -- emits 4-channel WAV
    that the speaker cannot replay correctly (STD-OUT is 2-slot).
  - `audio.new_input(mic_dev, 24000, 1, 16, 37.5)` -- omits `hw_channels=4`
    so the I2S TDM controller is misconfigured (total_slot=2) and the
    captured audio is garbled.
