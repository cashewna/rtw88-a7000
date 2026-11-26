# rtw88-a7000 Patch Stack

Local fork of [`lwfinger/rtw88`](https://github.com/lwfinger/rtw88) carrying targeted fixes for the **Netgear Nighthawk A7000** (`0846:9054`, Realtek RTL8814AU, `rfe_option = 1`) on Linux >= 6.9.

## Patches on top of `upstream/master`

1. **`fix(rtw8814a): initialize 2.4 GHz RFE state after power-on`** (`rtw8814a.c`)
   - `rtw8814a_table.c` (`bb_tbl`) initializes `REG_RFE_PINMUX_A..D` (`0x0cb0`, `0x0eb0`, `0x18b4`, `0x1ab4`) to 5 GHz RFE0 defaults (`0x54775477`) while `REG_CCK_CHECK` (`0x0454`) stays `0`.
   - When tuning to a 2.4 GHz channel directly after power-on or IPS, `rtw8814a_set_channel()` reads `REG_CCK_CHECK == 0`, concludes `old_band == RTW_BAND_2G == new_band`, and skips `rtw8814a_switch_band(RTW_BAND_2G)`.
   - Calling `rtw8814a_switch_band(rtwdev, RTW_BAND_2G, RTW_CHANNEL_WIDTH_20)` at the end of `rtw8814a_phy_set_param()` aligns the RFE pinmux (`0x77777777` for `rfe_option = 1`) and baseband with `REG_CCK_CHECK == 0`.

2. **`feat(rtw88): add active monitor mode and raw TX injection support`** (`main.h`, `main.c`, `mac80211.c`, `rx.c`, `tx.c`)
   - Advertises `NL80211_IFTYPE_MONITOR` and `NL80211_FEATURE_ACTIVE_MONITOR` (sets `hw->max_rate_tries = 63`).
   - Handles `NL80211_IFTYPE_MONITOR` vif lifecycle and `IEEE80211_CONF_CHANGE_MONITOR` filter/channel updates in `mac80211.c`.
   - Guards `hw->conf.chandef.chan` in `rx.c`.
   - Adds `rtw_tx_injected_pkt_info_update()` for `IEEE80211_TX_CTL_INJECTED` and control frames in `tx.c`, and skips `rtw_txq_check_agg()` for injected frames.

3. **`feat(rtw88): add monitor_channel debugfs knob for kernels >= 6.9`** (`debug.c`, `main.c`, `main.h`, `rx.c`)
   - Adds `/sys/kernel/debug/ieee80211/<phy>/rtw88/monitor_channel` (default `0`, mac80211-controlled).
   - Writing a channel number (e.g. `6` or `157`) locks the RF to that 20 MHz channel even if an unassociated STA vif is kept up for hardware auto-ACK alongside a monitor vif; writing `0` restores normal mac80211 control.
   - Reports `hal.current_channel` in `rx_status` while `mon_chan` is active so radiotap frequencies stay accurate.

## Runtime notes

- **USB 3.0 vs 2.4 GHz interference:** `rtw_usb` defaults to `switch_usb_mode=y` (SuperSpeed USB 3.0). If 2.4 GHz reception suffers from USB 3.0 RF noise on a particular port, load with `options rtw_usb switch_usb_mode=n` (forces USB 2.0) or use a shielded USB extension cable.
- **Forcing monitor channel with an active STA vif:**
  ```sh
  echo 6 | sudo tee /sys/kernel/debug/ieee80211/phy1/rtw88/monitor_channel
  # restore normal mac80211 control:
  echo 0 | sudo tee /sys/kernel/debug/ieee80211/phy1/rtw88/monitor_channel
  ```

## Rebasing & updating NixOS

```sh
cd /home/kk/Projects/rtw88-a7000
git fetch upstream
git rebase upstream/master

cd /etc/nixos
nix flake update rtw88-a7000
```
