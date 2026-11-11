# f407-hil-bench (Türkçe özet)

STM32F407 Discovery için donanım-döngüde (HIL) test tezgâhı. `bench.py` kartı yükler, resetler, seri
porttan konuşur, LED'leri Android telefon kamerasıyla izler ve TOML test planlarını çalıştırır. Her komut
`--json` ile makinenin okuyacağı bir sonuç ve olası hata sebeplerini verir; böylece bir yapay zekâ ajanı da
tezgâhı kullanabilir. Bir test kartı kilitlerse ya da çökertirse araç kartı kendisi kurtarır.

Her test başarısız olabildiğini de göstermek zorunda: `--verify-can-fail` planın bilerek bozulmuş bir
kopyasını çalıştırır ve o kopya FAIL vermelidir.

Bu repo [tezgah-lab](https://github.com/aarmert/tezgah-lab) çatısının parçası. Diğerleri:
[c031-ab-bootloader-stress](https://github.com/aarmert/c031-ab-bootloader-stress) (bu `bench.py`'nin bir
kopyasını kullanır) ve [f407-camera-blink](https://github.com/aarmert/f407-camera-blink) (kamera ölçümü ve
telefon aracı).

## Sonuçlar (28.10.2026, gerçek kart)

| Plan | Ne kontrol ediyor | Sonuç |
|---|---|---|
| `smoke` | açılış mesajı, PING, VER, TICK aralığı, UID, LED, bilinmeyen komut | PASS, negatif kontrol FAIL |
| `crash_recovery` | bilerek HardFault, sonra sessizlik, resetten sonra tekrar cevap | PASS, negatif kontrol FAIL |
| `measure` | besleme gerilimi, çip sıcaklığı, kart saatinin PC saatine oranı | PASS, negatif kontrol FAIL |
| `camera_blink` | 1/2/4 Hz yanıp sönme ve sabit yanma, telefon kamerasıyla | ayrıntılar f407-camera-blink'te |

Raporlar `results/runs/` altında.

## Donanımsız deneme

```bash
uv run firmware/build.py                                   # arm-none-eabi-gcc gerekir
uv run bench.py --sim run smoke --verify-can-fail
uv run bench.py --json --sim send PING --expect '^PANG$'    # bilerek yanlış beklenti: FAIL örneği
```

Kablolama, komutlar, plan yazma, kamera aracı ve üretim istasyonu için [README.md](README.md).
