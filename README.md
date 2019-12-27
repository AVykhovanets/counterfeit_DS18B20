# Your DS18B20 temperature sensor is likely a fake, counterfeit, clone...
...unless you bought the chips directly from [Maxim Integrated](https://www.maximintegrated.com/en/products/sensors/DS18B20.html) (or Dallas Semiconductor in the old days) or an authorized distributor (DigiKey, RS, Farnell, Mouser, Conrad, etc.), or you took exceptionally good care purchasing waterproofed DS18B20 probes. We bought over 1000 "waterproof" probes or bare chips from more than 70 different vendors on ebay, AliExpress, and online stores in 2019. All of the probes bought on ebay and AliExpress contained counterfeit DS18B20 sensors, and almost all sensors bought on those sites were counterfeit.

> Author: Chris Petrich, 28 December 2019.
> License: CC BY.
> Source: https://github.com/cpetrich/counterfeit_DS18B20/

## Why should I care?
Besides ethical concerns, some of the counterfeit sensors actually do not work in parasitic power mode, have a high noise level, temperature offset outside the advertised ±0.5 °C band, do not contain an EEPROM, have bugs and unspecified failure rates, or differ in another unknown manner from the specifications in the Maxim datasheet. Clearly, the problems are not big enough to discourage people from buying probes on ebay, but it may be good to know the actual specs when the data are important or measurement conditions are difficult.

## What are we dealing with?
Definitions differ, but following AIR6273, a **counterfeit** is an unauthorized copy, imitation, substitute, or modification misrepresented as a specific genuie item from an authorized manufacturer \[13\]. As of 2019, the main problem is imitations (**clones**) that are labeled to mislead the unsuspecting buyer. Fortunately, DS18B20 clones are nearly trivially easy to identify: Marking on the chip printed rather than lasered? No mark in the rear indent? Probably a conterfeit. Content of the "scratchpad register" doesn't comply with the datasheet? Probably a counterfeit. Behaves systematically different from known authentic chips? Probably a counterfeit.

## What do they look like?
![Authentic Maxim DS18B20 with topmark DALLAS DS18B20 1932C4 +786AB and indent marked "P"](images/Maxim_DS18B20_chip_front_reverse.jpg)

Above is an example of an **authentic**, Maxim-produced DS18B20 sensor in TO-92 case. 
* As of writing (2019), the topmark of original Maxim chips is lasered rather than printed. 
* The first two rows, ``DALLAS 18B20``, specify that this part is a DS18B20 (Dallas Semiconductor being the original producer),
* the ``+`` in the 4th row indicates that the part is RoHS compliant (\[1\]). 
* The 3rd row specifies production year and week number of the year (in this case, week 32 of 2019), and 
* the last two characters in row 3 specify the revision of the die (currently ``C4``). 
* In row 4, the three-digit number followed by two characters are a form of batch code that allows Maxim to trace back the production history. 
	+ In chips produced 2016 or later I've only come across character combinations ``AB`` and ``AC`` \[5\].
* The marking inside the indent on the rear of the case is
	+ ``P`` (Philippines?) on all recent chips (2016 and younger), and on most(?) chips going back at least as far as 2009 \[5\].
	+ ``THAI <letter>`` (Thailand?) where ``<letter>`` is one of ``I``, ``L``, ``N``, ``O``, ``S``, ``V``, and possibly others, at least on some chips produced around 2011 \[5\].
* From what I've seen, there is exactly one batch code associated with a date code for chips marked ``P`` in the indent \[5\]. This does not hold true for chips marked ``Thai`` in the indent \[5\].

## How do I know if I am affected?
If the DS18B20 have been bought from authorized dealers though a controlled supply chain then the chips are legit.

Otherwise, (I) one can test for compliance with the datasheet. If a sensor fails any of those tests, it is a fake (unless Maxim's implementation is buggy \[4\]). (II) one can compare sensor behavior with the behavior of Maxim-produced DS18B20. Those tests are based on the conjecture that all Maxim-produced DS18B20 behave alike. This should be the case at least for sensors that share a die code (which has been ``C4`` most likely since 2006 (definitely since 2009 \[5\])) \[5\].

Regarding (I), discrepancy between what the current datasheet says should happen and what the sensors do include \[1,5\]
* Family B: reserved bytes in scratchpad register can be overwritten (by following instructions in the datasheet)
* Family C: the sensor is fixed in 12-bit mode (i.e., byte 4 of the scratchpad register is always ``0x7f``)
* Family C: the number of EEPROM write cycles is very small (order of 10 rather than >50k)
* Family B2, D: significant number of sensors with offsets outside the ±0.5 C range at 0 °C
* Family D: sensor does not respond in parasitic mode (applies to most sensors of Family D)
* Family D: the temperature reading right after power-up is 25 rather than 85 °C
* Family D: sensor does not perform low-resolution temperature conversions faster
* Family D: reserved bytes 5 and 7 of the scratchpad register are not ``0xff`` and ``0x10``, respectively
* Family D1: retains temperature measurements during power cycles

Hence, as of writing (2019), every fake sensor available does not comply with the datasheet in at least one way.

Regarding (II), there are simple tests for differences with Maxim-produced DS18B20 sensors (``C4`` die) that apparently *all* counterfeit sensors fail \[5\]. The most straight-forward software tests are probably these:
1. It is a fake if its ROM address does not follow the pattern 28-xx-xx-xx-xx-00-00-xx \[5\]. (Maxim's ROM is essentially a 48-bit counter with the most significant bits still at 0 \[5\].)
2. It is a fake if ``<byte 6> == 0`` or ``<byte 6> > 0x10`` in the scratchpad register, or if the following scratchpad register relationship applies after **any** *successful* temperature conversion: ``(<byte 0> + <byte 6>) & 0x0f != 0`` (*12-bit mode*) \[3,5\].
3. It is a fake if the chip returns data to simple queries of undocumented function codes other than 0x68 and 0x93 \[4,5\]. (*As of writing (2019), this can actually be simplified to: it is a fake if the return value to sending code 0x68 is ``0xff`` \[5\].*)

In addition to obvious implementation differences such as those listed above under (I) and (II), there are also side-channel data that can be used to separate implementations. For example, the time reported for a 12 bit-temperature conversion (as determined by polling for completion after function code 0x44 at room temperature) is characteristic for individual chips (reproducible to much better than 1% at constant temperature) and falls within distinct ranges determined by the circuit's internals \[5\]:
* 11 ms: Family D1
* 28-30 ms: Family C
* 460-525 ms: Family D2
* 580-615 ms: Family A
* 585-730 ms: Family B

Hence, there will be some edge cases between Families A and B, but simply measuring the time used for temperature conversion will often be sufficient to determine if a sensor is counterfeit.

An important aspect for operation is a sensor's ability to pull the data line low against the fixed pull-up resistor. Turns out this abilitly differs between families. The datasheet guarantees that a sensor is able to sink at least 4 mA at 0.4 V at any temperature up to 125 °C \[1\]. Providing a current of 4 mA (1.2 kOhm pull-up resistor against 5 V), the following ``low`` voltages were achieved by the sensors at room temperature (note that only 5 to 10 sensors were measured per Family):
* Family A: 0.058 - 0.062 V
* Family B2: 0.068 - 0.112 V (all but one sensor: 0.068 - 0.075 V)
* Family C: 0.036 - 0.040 V
* Family D2: 0.121 - 0.124 V

All sensors are well within specs at room temperature but clustering of data by Family is apparent, indicating that the hardware was designed independently. It could be interesting to repeat these measurements above 100 °C.

Alternatively,
* It is a fake if the date--batch combination printed on the case of the sensor is not in the Maxim database (need to ask Maxim tech support to find out).

Note that none of the points above give certainty that a particular DS18B20 is an authentic Maxim product, but if any of the tests above indicate "fake" then it is most defintely counterfeit \[5\]. Based on my experience, a sensor that will fail any of the three software tests will fail all of them.

## What families of DS18B20-like chips can I expect to encounter?
Besides the DS18B20 originally produced by Dallas Semiconductor and continued by Maxim Integrated after they purchased Dallas (Family A, below), similar circuits seem to be produced independently by at least 3 other companies in 2019 (Families B, C, D) \[5\]. The separation into families is based on patterns in undocumented function codes that the chips respond to as similarities at that level are unlikely to be coincidental \[5\]. Chips of Family B seem to be produced by [Beijing 7Q Technology](http://www.7qtek.com).

In our ebay purchases in 2018/19 of waterproof DS18B20 probes from China, Germany, and the UK, most lots had sensors of Family B1, while one in three purchases had sensors of Family D. None had sensors of Family A or C. Neither origin nor price were indicators of sensor Family. When purchasing DS18B20 chips, Family D was clearly dominant with Family B2 coming in second, and a small likelihood of obtaining chips of Families C or A.

In the ROM patterns below, *tt* and *ss* stand for fast-changing and slow-changing values within a production run \[5\], and *crc* is the CRC8 checksum defined in the datasheet \[1\].

### Family A: Authentic Maxim DS18B20
***Obtained no probes containing these chips on ebay or AliExpress in 2019, but obtained chips from a few vendors in 2019***
* ROM pattern \[5\]: 28-tt-tt-ss-ss-00-00-crc
* Scratchpad register:  ``(<byte 0> + <byte 6>) & 0x0f == 0`` after all successful temperature conversions, and ``0x00 < <byte 6> <= 0x10`` \[2,3,5\].
* According to current behavior \[5\] and early datasheets \[9\], the power-up state of reserved ``<byte 6>`` in the Scratchpad register is ``0x0c``.
* Returns "Trim1" and "Trim2" values if queried with function codes 0x93 and 0x68, respectively \[4\]. The bit patterns are very similar to each other within a production run \[4\]. Trim2 is currently less likely to equal 0xff than Trim1 \[5\]. Trim2 was 0xDB or 0xDC since at least 2009, and has been 0x74 since the fall of 2016 (all with ``C4`` die) \[5\].
	+ Trim1 and Trim2 encode two parameters \[5\]. Let the bit pattern of Trim1 be ``[t17, t16, t15, t14, t13, t12, t11, t10]`` (MSB to LSB) and Trim2 be ``[t27, t26, t25, t24, t23, t22, t21, t20]``. Then,
		- offset parameter = ``[t22, t21, t20, t10, t11, t12, t13, t14, t15, t16, t17]`` (unsigned 11 bit-value) \[5\], and
		- curve parameter = ``[t27, t26, t25, t24, t23]`` (unsigned 5 bit-value) \[5\].
	+ Within a batch, the offset parameter seems to spread over 20 to 30 units while all sensors within the batch share the same curve parameter \[5\].
	+ The offset parameter shifts the temperature output over a range of approx. 100 °C (0.053 °C per unit), while the curve parameter shifts the temperature over a range of 3.88 °C (0.12 °C per unit), at least in current versions of the chip \[5\]. Example values of 2019 are ``offset = 0x420`` and ``curve = 0x0E``, i.e. they lie pretty central within their respective ranges.
* Temperature offset of current batches (2019) is as shown on the [Maxim FAQ](https://www.maximintegrated.com/en/support/faqs/ds18b20-faq.html) page, i.e. approx. +0.1 °C at 0 °C \[6\] (*i.e., not as shown on the datasheet \[1,9\]. The plot on the datasheet stems from measurements at the time of introduction of the sensor 10+ years ago \[5,10\].*). Very little if any temperature discretization noise \[5\].
* Polling after function code 0x44 indicates a spread of 584-615 ms between sensors for a 12-bit temperature conversion at room temperature \[5\]. Conversion time is easily repeatable for individual chips. Lower resolutions cut the time in proportion, i.e. 11 bit-conversions take half the time. The trim parameters affect the conversion time.
* It appears the chip returns a temperature of 127.94 °C (=0x07FF / 16.0) if a temperature conversion was unsuccessful \[5\] (e.g. due to power stability issues which arise reproducibly in "parasitic power" mode with *multiple* DS18B20 if Vcc is left floating rather than tied to ground. Note that the datasheet clearly states that Vcc is to be tied to GND in parasitic mode.).

- Example ROM: 28-13-9B-BB-0B **-00-00-** 1F
- Initial Scratchpad: **50**/**05**/4B/46/**7F**/**FF**/0C/**10**/1C
- Example topmark: DALLAS DS18B20 1932C4 +786AB
- Indent mark: ``P`` (date codes 1150 and newer)

### Family A-Fishy1: Stolen?
***Obtained no probes containing these chips on ebay or AliExpress in 2019, but obtained chips from one vendor in 2019***

*If I were to make a wild guess I would say these chips were diverted somewhere toward the end of the Maxim production pipeline (stolen?) \[5\].* These chips are marked as produced in Thailand rather than Philippines.

* ROM pattern \[5\]: 28-tt-tt-Cs-03-00-00-crc

The chips follow the description of Family A above with the following exceptions \[5\]:
* Both alarm registers are set to 0x00 (scratchpad bytes 2 and 3).
* The conversion resolution is set to 9 bits (i.e., both configuartion bits are 0).
* Both trim values are 0x00, resulting in wrong temperatures (i.e., very low) and conversion times in the range of 400 to 500 ms.
	+ Once trim values are set to something reasonable, the time for temperature conversion is within the range specified for Family A above.

- Example ROM: 28-9B-9E-CB-03 **-00-00-** 1F
- Initial Scratchpad: **50**/**05**/00/00/**1F**/**FF**/0C/**10**/74
- Example topmark: DALLAS DS18B20 1136C4 +957AE
- Example topmark: DALLAS DS18B20 1136C4 +957AF
- Example topmark: DALLAS DS18B20 1136C4 +152AE
- Example topmark: DALLAS DS18B20 1136C4 +152AF
- Example topmark: DALLAS DS18B20 1136C4 +152AG
- Example topmark: DALLAS DS18B20 1136C4 +152AI
- Indent mark: ``THAI <letter>``

### Family A-Fishy2: Stolen?
***Obtained no probes containing these chips on ebay or AliExpress in 2019, but obtained chips from one vendor in 2019***

*If I were to make a wild guess I would say the dies predate ``C4`` and that either the dies themselves were diverted or the masks used to produce the dies were stolen \[5\].*

* ROM pattern \[5\]: 28-00-ss-00-tt-tt-tt-crc, 28-ss-00-ss-tt-tt-tt-crc, 28-ss-00-00-tt-tt-00-crc

The chips follow the description of Family A above with the following exceptions \[5\]:
* The ROM pattern is incompatible with what Maxim produces.
* The Trim2 value is ``0xFB`` or ``0xFC``, i.e. incompatible with a known \[5\] Maxim production suggested by the date code. (Note that this means the curve parameter is 0x1f, i.e. the highest value possible \[5\]. Also, the offset parameter spreads over 200 units rather than a range typical for Family A, see above \[5\].)
* The time for temperature conversion spans a remarkably wide range from 325 to 502 ms. This range remains wide and outside the bounds of Family A specified above even when applying more recent trim settings.
* Alarm settings (i.e., scratchpad bytes 2 and 3) have seemingly random content.
* *Some* chips retain their scratchpad content across a 100 ms power cycle.
* One specimen tested did not function properly in parasitic mode.
* The topmark is printed rather than lasered, and there is no mark in the indent.

- Example ROM: 28-19-00-00-B7-5B-00-41
- Initial Scratchpad: **50**/**05**/xx/xx/**7F**/**FF**/0C/**10**/xx
- Example topmark: DALLAS DS18B20 1808C4 +233AA
- Indent mark: *none*
	
### Family B1: QT18B20 Matching Datasheet Temperature Offset Curve
***Obtained probes from a number of vendors in 2019, obtained chips from one vendor in 2019***
* ROM patterns \[5\]:
	- 28-AA-tt-ss-ss-ss-ss-crc
	- 28-tt-tt-ss-ss-ss-ss-crc
* Scratchpad register ``<byte 6>`` is constant (default ``0x0c``) \[5\].
* DS18B20 write scratchpad-bug (0x4E) / QT18B20 scratchpad \[5,12\]:
	- If 3 data bytes are sent (as per DS18B20 datasheet, TH, TL, Config) then ``<byte 6>`` changes to the third byte sent,
	- if 5 data bytes are sent (as per QT18B20 datsheet, TH, TL, Config, User Byte 3, User Byte 4), the last two bytes overwrite ``<byte 6>`` and ``<byte 7>``, respectively.
* Does not return data on undocumented function code 0x68 \[5\]. Does return data from codes 0x90, 0x91, 0x92, 0x93, 0x95, and 0x97 \[5\]. Return value in response to 0x97 is ``0x22`` \[5\].
* ROM code can be changed in software with command sequence "96-Cx-Dx-94" \[5\].
* Temperature offset as shown on the datasheet (-0.15 °C at 0 °C) \[6\]. Very little if any temperature discretization noise \[5\].
* Polling after function code 0x44 indicates approx. 589-728 ms for a 12-bit temperature conversion and proportionally less at lower resolution \[5\].

- Example ROM: 28 **-AA-** 3C-61-55-14-01-F0
- Example ROM: 28-AB-9C-B1 **-33-14-01-** 81
- Initial Scratchpad: 50/05/4B/46/7F/FF/0C/10/1C
- Example topmark: DALLAS 18B20 1626C4 +233AA
- Example topmark: DALLAS 18B20 1810C4 +051AG
- Indent mark: *none*

### Family B2: QT18B20 with -0.5 °C Temperature Offset at 0 °C
***Obtained both probes and chips of this series from a number of vendors in 2019. Two vendors sent chips marked 7Q-Tek rather than DALLAS***
* ROM patterns \[5\]: 28-FF-tt-ss-ss-ss-ss-crc
* Scratchpad register ``<byte 6>`` is constant (default ``0x0c``) \[5\].
* DS18B20 write scratchpad-bug (0x4E) / QT18B20 scratchpad \[5,12\]:
	- If 3 data bytes are sent (as per DS18B20 datasheet, TH, TL, Config) then ``<byte 6>`` changes to the third byte sent,
	- if 5 data bytes are sent (as per QT18B20 datsheet, TH, TL, Config, User Byte 3, User Byte 4), the last two bytes overwrite ``<byte 6>`` and ``<byte 7>``, respectively.
* Does not return data on undocumented function code 0x68 \[5\]. Does return data from codes 0x90, 0x91, 0x92, 0x93, 0x95, and 0x97 \[5\]. Return value in response to 0x97 is ``0x31`` \[5\].
* ROM code can **not** be changed in software with command sequence "96-Cx-Dx-94" \[5\].
* Typical temperature offset at at 0 °C is -0.5 °C \[6\]. Very little if any temperature discretization noise \[5\].
* Polling after function code 0x44 indicates approx. 587-697 ms for a 12-bit temperature conversion and proportionally less at lower resolution \[5\].

- Example ROM: 28 **-FF-** 7C-5A-61-16-04-EE
- Initial Scratchpad: 50/05/4B/46/7F/FF/0C/10/1C
- Example topmark: DALLAS 18B20 1626C4 +233AA
- Example topmark: DALLAS 18B20 1702C4 +233AA
- Example topmark: DALLAS 18B20 1810C4 +138AB
- Example topmark: DALLAS 18B20 1829C4 +887AB
- Example topmark: DALLAS 18B20 1832C4 +827AH
- Example topmark: DALLAS 18B20 1908C4 +887AB
- Example topmark: 7Q-Tek 18B20 1861C02
- Indent mark: *none*

### Family C: Small Offset at 0 °C
***Obtained no probes but obtained chips from a few vendors in 2019***
* ROM patterns \[5\]: 28-FF-64-ss-ss-tt-tt-crc
* Scratchpad register ``<byte 6> == 0x0c`` \[5\].
* Does not return data on undocumented function code 0x68 or any other undocumented function code \[5\].
* Typical temperature offset at 0 °C is +0.05 °C \[6\]. Very little if any temperature discretization noise \[5\].
* EEPROM endures only about eight (8) write cycles (function code 0x48) \[5\].
* Polling after function code 0x44 indicates 28-30 ms (thirty) for a 12-bit temperature conversion \[5\]. Temperature conversion works also in parasite power mode \[5\].
* Operates in 12-bit conversion mode, only (configuration byte reads ``0x7f`` always) \[5\].
* Default alarm register settings differ from Family A (``0x55`` and ``0x00``) \[5\].

- Example ROM: 28 **-FF-64-** 1D-CD-96-F2-01
- Initial Scratchpad: 50/05/55/00/7F/FF/0C/10/21
- Example topmark: DALLAS 18B20 1331C4 +826AC
- Example topmark: DALLAS 18B20 1810C4 +158AC
- Indent mark: *none*

### Family D1: Noisy Rubbish with Supercap
***Obatined probes from two vendors in early 2019, obtained chips from one vendor in 2019***
* ROM patterns \[5\]: 28-tt-tt-77-91-ss-ss-crc and 28-tt-tt-46-92-ss-ss-crc
* Scratchpad register ``<byte 7> == 0x66``, ``<byte 6> != 0x0c`` and ``<byte 5> != 0xff`` \[5\].
* Does not return data on undocumented function code 0x68 \[5\]. Responds back with data or status information after codes 
	+ 0x4D, 0x8B (8 bytes), 0xBA, 0xBB, 0xDD (5 bytes), 0xEE (5 bytes) \[5\], or
	+ 0x4D, 0x8B (8 bytes), 0xBA, 0xBB \[5\].
* First byte following undocumented function code 0x8B is \[5\]
	+ ``0x06``: Sensors **do not work with Parasitic Power**. Sensors leave data line floating when powered parasitically \[5\].
	+ ``0x02``: Sensors do work in parasitic power mode (and report correctly whether they are parasitically powered).
* It is possible to send arbitrary content for ROM code and for bytes 5, 6, and 7 of the status register after undocumented function codes 0xA3 and 0x66, respectively \[5\].
* Temperature errors up to 3 °C at 0 °C \[6\]. Very noisy data \[5\].
* Polling after function code 0x44 indicates approx. 11 ms (eleven) for conversion regardless of measurement resolution \[5\].
* Chips **contain a supercap rather than an EEPROM** to hold alarm and configuration settings \[5\]. I.e., the last temperature measurement and updates to the alarm registers are retained between power cycles that are not too long \[5\].
	+ The supercap retains memory for several minutes unless Vcc is pulled to GND, in which case memory retention is 5 to 30 seconds \[5\].
* Initial temperature reading is 25 °C or the last reading before power-down \[5\]. Default alarm register settings differ from Family A (``0x55`` and ``0x05``) \[5\].

- Example ROM: 28-48-1B-77 **-91-** 17-02-55  (working parasitic power mode)
- Example ROM: 28-24-1D-77 **-91-** 04-02-CE  (responds to 0xDD and 0xEE)
- Example ROM: 28-B8-0E-77 **-91-** 0E-02-D7
- Example ROM: 28-21-6D-46 **-92-** 0A-02-B7
- Initial Scratchpad: 90/01/55/05/7F/7E/81/66/27
- Example topmark: DALLAS 18B20 1807C4 +051AG
- Example topmark: DALLAS 18B20 1827C4 +051AG
- Indent mark: *none*

### Family D2: Noisy Rubbish
***Obtained both probes and chips from a large number of vendors in 2019***
* ROM patterns \[5\]: 28-tt-tt-79-97-ss-ss-crc, 28-tt-tt-94-97-ss-ss-crc, 28-tt-tt-79-A2-ss-ss-crc, 28-tt-tt-16-A8-ss-ss-crc
* Scratchpad register ``<byte 7> == 0x66``, ``<byte 6> != 0x0c`` and ``<byte 5> != 0xff`` \[5\].
* Does not return data on undocumented function code 0x68 \[5\]. Responds back with data or status information after codes 
	+ 0x4D, 0x8B (9 bytes), 0xBA, 0xBB, 0xDD (3 bytes), 0xEE (3 bytes) \[5\], or
	+ 0x4D, 0x8B (9 bytes), 0xBA, 0xBB \[5\].
* First byte following undocumented function code 0x8B is ``0x00`` \[5\].
* Sensors **do not work with Parasitic Power**. Sensors draw data line low while powered parasitically \[5\].
* Temperature errors up to 3 °C at 0 °C \[6\]. Data noisier than genuie chips \[5\].
* Polling after function code 0x44 indicates approx. 462-523 ms for conversion regardless of measurement resolution \[5\]. The series with ``97`` and ``A2``/``A8`` in the ROM converts in 494-523 ms and 462-486 ms, respectively \[5\]. Chips with ``A2`` or ``A8`` in byte 4 of the ROM seem to have appeared first in 2019.
* Initial temperature reading is 25 °C \[5\]. Default alarm register settings differ from Family A (``0x55`` and ``0x05``) \[5\].

- Example ROM: 28-90-FE-79 **-97-** 00-03-20
- Example ROM: 28-FD-58-94 **-97-** 14-03-05
- Example ROM: 28-FB-10-79 **-A2-** 00-03-88
- Example ROM: 28-29-7D-16 **-A8-** 01-3C-84
- Initial Scratchpad: 90/01/55/05/7F/xx/xx/66/xx
- Example topmark: DALLAS 18B20 1812C4 +051AG
- Example topmark: DALLAS 18B20 1827C4 +051AG
- Example topmark: DALLAS 18B20 1916C4 +051AG
- Example topmark: DALLAS 18B20 1923C4 +051AG
- Indent mark: *none*

### Obsolete as of 2019
***Obtained neither probes nor chips in 2019***
* ROM patterns \[5,7\]: 28-tt-tt-ss-00-00-80-crc
	- Example ROM: 28-9E-9C-1F **-00-00-80-** 04
* ROM patterns \[5,11\]: 28-61-64-ss-ss-tt-tt-crc
	- Example ROM: 28 **-61-64-** 11-8D-F1-15-DE

## 7Q-Tek QT18B20
The QT18B20 is a DS18B20 clone developed and sold by Beijing 7Q Technology Inc (Family B). The datasheet of the QT18B20 emphasizes the addition of two user-defined bytes in the scratchpad register \[12\]. Unlike the data sheet of the DS18B20, it does not state that the ROM code is lasered. A large number of these chips bear fake DS18B20 topmarks.

## MAX31820
The MAX31820 is a DS18B20 with limited supply voltage range (i.e. up to 3.7 V) and smaller temperature range of high accuracy \[1,8\]. Like the DS18B20, it uses one-wire family code 0x28 \[1,8\]. Preliminary investigations have not (yet) revealed a test to distinguish between DS18B20 of Family A and Maxim-produced MAX31820 in software \[5\].

## Warning
**Sending undocumented function codes to a DS18B20 sensor may render it permanently useless,** for example if temperature calibration coefficients are overwritten \[5\]. The recommended (and currently sufficient) way of identifying counterfeit sensors is to analyze state and behavior of the scratchpad register in response to commands that comply with the datasheet \[5\].


(*Information on chips of Families A, B, C, and D comes from my own investigations of sensors in conjunction with the references below as indicated by reference number \[1-6,8-10\]. Tests were performed at 5 V with 1.2 kOhm pull-up.*)

## References

1. [DS18B20](https://datasheets.maximintegrated.com/en/ds/DS18B20.pdf) "DS18B20 Programmable Resolution 1-Wire Digital Thermometer", Datasheet 19-7487 Rev 6 7/19, Maxim Integrated.
2. [DS18S20](https://datasheets.maximintegrated.com/en/ds/DS18S20.pdf) "DS18S20 High-Precision 1-Wire Digital Thermometer", Datasheet, Maxim Integrated.
3. [AN4377](https://www.maximintegrated.com/en/design/technical-documents/app-notes/4/4377.html) "Comparison of the DS18B20 and DS18S20 1-Wire Digital Thermometers", Maxim Integrated
4. AN247 "DS18x20 EEPROM Corruption Issue", Maxim Integrated
5. Own investigations 2019, unpublished.
6. Petrich, C., M. O'Sadnick, Ø. Kleven, I. Sæther (2019). A low-cost coastal buoy for ice and metocean measurements. In Proceedings of the 25th International Conference on Port and Ocean Engineering under Arctic Conditions (POAC), Delft, The Netherlands, 9-13 June 2019, 6 pp.
7. Contribution of user *m_elias* on https://forum.arduino.cc/index.php?topic=544145.15
8. [MAX31820](https://datasheets.maximintegrated.com/en/ds/MAX31820.pdf) "1-Wire Ambient Temperature Sensor", Datasheet, Maxim Integrated.
9. DS18B20 "DS18B20 Programmable Resolution 1-Wire Digital Thermometer", Datasheet 043001, Dallas Semiconductor, 20pp.
10. DS18B20 "DS18B20 Programmable Resolution 1-Wire Digital Thermometer", Preliminary Datasheet 050400, Dallas Semiconductor, 27pp.
11. Piecemeal from various blogs and posts.
12. [QT18B20](http://www.leoniv.diod.club/articles/ds18x20/downloads/qt18b20.pdf) "QT18B20 Programmable Resolution 1-Wire Digital Thermometer", Datasheet Rev 061713, 7Q Technology.
13. [AIR6273](https://www.sae.org/standards/content/air6273/) "Terms, Definitions, and Acronyms Counterfeit Materiel or Electrical, Electronic, and Electromechanical Parts", SAE Aerospace Information Report, July 2019.
