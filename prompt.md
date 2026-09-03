Fix this Omarchy machine so a room HDMI / Crestron / projector input actually shows this computer.

Symptom to treat as the bug: Hyprland lists the sink (EDID present, often "Crestron") but the room panel / projector input list does not light up this machine, or the screen stays black. Do not stop at "hyprctl monitors says it's on."

Follow the Omarchy skill. Edit only ~/.config/hypr/ and user-owned helpers. Never edit /usr/share/omarchy/. After any Hyprland Lua change: hyprctl reload && hyprctl configerrors until clean. Hyprland 0.55+ is Lua: hyprctl dispatch uses hl.dsp.* (example: hl.dsp.dpms({action = "enable", monitor = "DP-5"})). hyprctl keyword monitor is legacy and may fail.

Do this in order:

1. Identify the sink
   hyprctl monitors all
   for f in /sys/class/drm/card*-*/status; do echo "$f $(cat $f)"; done
   journalctl -k --since '15 min ago' | rg -i 'amdgpu|i915|link|stream|hdmi|dp-'
   Record connector name, description, current mode, scale, mirrorOf, dpms.

2. Decide which failure you have
   A. Empty extended desktop: output is live but workspace has no windows. For a projector, enable stock mirroring:
      omarchy-hyprland-monitor-internal-mirror on
   B. No TMDS / switcher blind: dmesg has "enabling link N failed: 15" and/or "Adding stream ... failed with err 28". Linux read EDID; the GPU did not train a video link. This is the Crestron/HDMI-adapter case.

3. Pin a switcher-safe mode in ~/.config/hypr/monitors.lua
   Last matching hl.monitor() wins. Add a named rule after the catch-all. Use the hyprctl description with a desc: prefix (no port name).
   Typical Crestron / room HDMI:
     mode = "1920x1080@60"   -- CEA, 148.5 MHz. Prefer this over "preferred" if the catch-all is scaled.
     scale = 1
     bitdepth = 8
     cm = "srgb"
     vrr = 0
     supports_hdr = -1
     supports_wide_color = -1
     mirror = "<laptop output>"   -- usually eDP-1; use omarchy-hyprland-monitor-laptop
   Do not apply the laptop's scale-2 catch-all to a 1080p sink.

4. If the GPU is AMD, fix DP link rate (root, once per plug — never on a timer)
   Read:
     /sys/kernel/debug/dri/*/<CONNECTOR>/link_settings
   Example of the bug:
     Current:  4  0x1e  0   (HBR3 — driver preferred)
     Verified: 4  0x14  16  (HBR2 — all this sink trained)
   Rates: 0x06 RBR, 0x0a HBR, 0x14 HBR2, 0x1e HBR3.
   If Current rate > Verified rate, write the verified lane+rate:
     printf '4 0x14 0\n' > .../link_settings
   Writing retrains. Do not write HBR3 onto HDMI adapters / Crestron / Samsung HDMI dongles — that leaves them black.
   Hyprland reload resets Preferred to HBR3. After any modeset, write Verified again. Apply, sleep 2, apply again.
   Persist a oneshot udev+systemd helper (not a 1s loop). If this machine already has /usr/local/sbin/omarchy-studio-display-hbr3, extend it: Studio Display → HBR3; any other connected DP whose Current > Verified → write Verified. Install with the existing install.sh + pkexec. If it does not exist, create an equivalent HDMI-only helper; do not invent HBR3 for HDMI.

5. Remake without blowing the rate
   Prefer: write Verified, then hl.dsp.force_renderer_reload(), then write Verified again.
   Avoid trigger_hotplug and hyprctl reload while diagnosing — both remake at HBR3 and reintroduce err 28.
   DPMS cycle is ok if you immediately re-pin Verified:
     hyprctl dispatch 'hl.dsp.dpms({action = "disable", monitor = "<CONN>"})'
     sleep 2
     hyprctl dispatch 'hl.dsp.dpms({action = "enable", monitor = "<CONN>"})'

6. Optional extras (do not treat as the fix)
   CEC: cec-ctl --device=/dev/cec0 --osd-name=<hostname> --playback
        cec-ctl --device=/dev/cec0 --to 0 --image-view-on
        cec-ctl --device=/dev/cec0 --active-source phys-addr=1.0.0.0
   HDMI audio: play a short tone to the AMD HDMI sink so audio infoframes exist.
   HDCP: many Crestron inputs advertise HDCP 1.4/2.2. Linux often cannot enable it (Content Protection stays Undesired). If video is trained and the panel is still dark, say so — that is room policy, not a modeset bug.

7. Prove it from evidence, then ask the human to look at the panel
   Must be true:
   - link_settings Current == Verified (not HBR3-above-HBR2)
   - no new "enabling link N failed: 15" / err 28 after the pin
   - hyprctl: 1920x1080@60, scale 1, dpms on, mirroring the laptop if this is a projector
   - AMD DTN (if present): HUBP clock_en=1 blank_en=0, L_ENC DP_LINK_TRAINING_COMPLETE=1 on that link
   Done only when the human says the projector / Crestron input shows this machine.

Safety on Omarchy / this hardware family:
- Do not set AQ_DRM_DEVICES to /dev/dri/by-path/pci-… (colons → zero GPUs → SDDM loop). Use /dev/dri/card1:/dev/dri/card2 only.
- Do not run sudo in a loop. pkexec for debugfs/udev install.
- Do not write link_settings on a timer.
- Do not force HBR3 on non-Studio Display sinks.
- Super+Ctrl+Alt+Delete toggles laptop mirroring if you used the stock helper.
