# Hazardous Environment Air Interface

HEAI is an in-progress GSM PHY/MAC intended to function as a subcomponent of a GSM cell modem, designed to enable development of upper protocol layers with [LANGSEC](http://langsec.org/) principles firmly in mind. Desired characteristics include a strong separation between DSP-related tasks and protocol parsing/state as follows:


## Architecture
* All DSP/SDR functions live in the FPGA, which has an I/Q interface with an RF transceiver and a low-speed serial interface with a host machine.

* All protocol state, message parsing, and message generation/serialisation/"unparsing" is done in an appropriately sandboxed process on the host machine.



## Reasoning
* The host machine does not handle real-time tasks, I/Q samples, nor DSP maths. The host treats the FPGA/RF components as "just another network interface" (albeit one using an abstruse protocol instead of TCP/IP/Ethernet) and *does not need a powerful, SIMD-capable CPU*. Similarly, the link between the FPGA and the host can be low bitrate, since it does not have to transport I/Q samples -- just bits of data frames. Offloading all the DSP-related tasks onto the FPGA will let us keep the host's CPU in low-power states most of the time, which is critical for operation with power/energy constraints.

* The FPGA gateware (and code executed on soft cores within the FPGA) does not manage protocol parsing/state. Rather, it concerns itself only with channel coding/decoding, modulation, demodulation, burst timing, and interfacing with the transmit/receive RF chains. We impose this constraint to reduce the amount and complexity of verification that needs to be done on the Verilog code.

* Doing all the protocol parsing/state tasks on the host machine means that we can trivially use the memory-safe/strongly-typed programming languages of our choice. Published vulnerabilities for modern cellular modems are disappointingly reminiscent of 1990s-vintage TCP/IP stack vulnerabilities, such as [a fragmentation reassembly bug leading to arbitrary code execution](https://comsecuris.com/blog/posts/theres_life_in_the_old_dog_yet_tearing_new_holes_into_inteliphone_cellular_modems/). I believe [LANGSEC](http://langsec.org/) techniques can help us do much better, and this design decision will make it easier to generate/test/fuzz/verify parsers/state-machines/unparsers for complex languages/protocols -- a critical step towards creating a *trustworthy* GSM MS.


## Caveats
* While our objective is to create a GSM MS that is secure even against a highly-competent adversary, the GSM protocol specifies broken cryptographic algorithms. This means that **no GSM implementation can provide privacy/integrity** for user data *or* control-plane signalling, and that even a perfect implementation will be inherently vulnerable to eavesdropping/denial-of-service/modification of transmitted/received data. Use a good VPN, like wireguard.

* There are no plans to do anything with CDMA/WCDMA/UMTS/HSPA, only GSM (and later, LTE).

* I haven't started implementing the receiver components yet.

## TODO/plan

* DSP Toolbox:
    1. GMSK modulator in python (arbitrary samples/symbol)
    2. COST207 channel models, with attenuation and noise
    3. framework to run arbitrary simulations for a given single channel model with different {phase offsets, freq offsets, timing offsets, bit sequences, noise amplitudes} and plot statistics of estimator accuracy and bit-error-rate vs Eb/N0

* Implement and evaluate performance of feedforward/data-aided estimators:
    1. data-aided midamble correlation to find symbol timing offset
    2. data-aided correlation to find freq/phase offsets (IDK if this will be necessary due to adaptive MLSE equalization, but we will do it if only for symbol-by-symbol demod)
    3. data-aided correlation to estimate channel impulse response

* Symbol-by-symbol demodulation as proof-of-concept:
    1. Investigate derotation (allegedly converts GMSK to BPSKish constellation)
    2. Check out performance of symbol-by-symbol demodulation of GMSK and also derotated GMSK (see if the derotation scheme worsens anything) with COST207 channel models

* Viterbi equalization/demodulation:
    1. Investigate how many samples per symbol Viterbi needs and matched-filtering
    2. Implement Viterbi equalizer components (ACS, trellis, filter, CIR update)


* Implementation concerns:
    1. write gateware to interface FPGA board with LMS6002D board (read/write to its SPI registers)
    2. UART command scheme (parsing and comman/data ringbuffers)
    3. Look into nmigen or clash for implementing DSP components of receiver

## Warning
The code in this repository is very incomplete. Do not use it for broadcasting RF unless you do appropriate conformance testing, since I haven't yet.
