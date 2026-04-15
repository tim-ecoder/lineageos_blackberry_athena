VoIP Audio Fix — Base patches (pre plan-b)

Enables non-compress VoIP path for Telegram/WhatsApp calls.
Compress-voip disabled due to ADSP incompatibility with incoming calls.

kernel.patch:
  q6asm.c: preprocopo_id=0 → ASM_STREAM_PREPROCOPO_ID_NONE (0x10C68)
  msm-pcm-routing-v2.c: topology=0 → app_type=0

hal-voice.patch:
  voice.c: VOIP_CALL in voice_update_devices (BT SCO reroute)

device-sdm660.patch:
  vendor.prop: compr_voip=false, pcm.voip=false, fluence.voicecomm=true
  audio_policy_configuration.xml: voip_rx + BT SCO routes

device-athena.patch:
  audio_platform_info.xml: USECASE_AUDIO_RECORD_VOIP → device 0
  mixer_paths_bbf100.xml: audio-record-voip path with boosted gain

Status: Outgoing VoIP works. Incoming VoIP has low-fi TX quality.
Next step: Plan-b (AOSP VoIP devices with MI2S backends)
