---
# You can also start simply with 'default'
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: "SUSS: Improving TCP Performance by Speeding Up Slow-Start"
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# apply unocss classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
---

# SUSS: Improving TCP Performance by Speeding Up Slow-Start

<br>

## Mahdi Arghavani, Haibo Zhang, David Eyers, and Abbas Arghavani *SIGCOMM 2024*


---
transition: fade-out
---

# TCP slow-start drawbacks
<div v-click>

- slow ramping-up of the data delivery rate
  - inefficient bandwidth utilization

</div>

<div v-click>

- prolonged completion time for small-size flows (flows less than a few megabytes in size)
  - significantly diminish the Quality of Experience (QoE)

</div>

<div v-click>

<img src="/fig1.png" style="margin: auto; max-width: 70%; max-height: 100%;">

</div>

<div v-click>

- **Two plots highlight the capability for transferring a larger amount of data during the initial phase**

</div>


---
transition: fade-out
---

# TCP slow-start drawbacks
<div v-click>

- Inefficiency of CUBIC's slow-start in a congested network path

</div>

<br>

<div v-click>

<img src="/fig2.png" style="margin: auto; max-width: 80%; max-height: 100%;">

</div>

<br>

<div v-click>

- CUBIC flow cannot gain a fair share of network resources rapidly, resulting in prolonged unfairness
  - reason: sensitivity in packet loss → exit slow-start before reaching $cwnd^*$

</div>


---
transition: fade-out
---


# Related Works

<div v-click>

- network-assisted approaches
  - require feedback from routers and switches → challenging for deployment and testing

</div>

<div v-click>

- End-to-end solutions
  - designed to estimate $c w n d^*$, based on bandwidth measurement and optimal rate estimation
  - accurate bandwidth measurement and rate estimation are very challenging, especially in early rounds

</div>

<div v-click>

- intuitive thoughts: use a larger initial window size (_iw_) → now 15KB (10 segments, RFC6928)
  - unaware of the flow length, lack prior knowledge of the network path conditions at the beginning
    - cannot select _iw_ appropriately
  - the consequences of releasing a large burst of data packets in early rounds

</div>

---
transition: fade-out
---

# SUSS

#### A lightweight, sender-side add-on to CUBIC's slow-start mechanism

<div v-click>

- Aims
  - safely accelerate cwnd growth in large-BDP networks
  - enhance the delivery rate during the early RTTs of slow-start

</div>

<div v-click>

- Key idea: predict the continuation of exponential growth of cwnd in the subsequent RTT
  - if _cwnd_ is expected to grow in the next RTT, then quadruple _cwnd_
  - allow _cwnd_ to quickly ramp up in the early RTTs of slow-start

</div>

<div v-click>

- primary challenges
  - How to predict the likelihood of exponential _cwnd_ growth in the next RTT
    - based on HyStart: _cwnd_ ≈ _$c w n d^*$_
  - How to control additional data packet transmission to avoid bursts and packet loss
    - propose a novel combination of <span v-mark.underline.orange="4">ACK clocking and packet pacing</span>
      - ACK clocking: protect the functionality of HyStart and its estimation of the growth factor
      - packet pasing: avoid the bursts of traffic that might be caused by surges in _cwnd_

</div>

---
transition: fade-out
---

# Key parameters

<div v-click>

<img src="/fig4.png" style="margin: left; max-width: 65%; max-height: 100%;">

</div>

---
transition: fade-out
---

# Theory Behind SUSS

<div v-click>

- After receive the ACK train: 
  - evaluate whether the exponential growth of _cwnd_ is extrapolated to persist for the next round
  - if true, then send another $2 * c w n d_{i-1}$ this round → $G_i = 4$
  - Why not consider more rounds? the network condition may vary over several upcoming rounds

</div>

<div v-click>

<img src="/fig5.png" style="margin: auto; max-width: 55%; max-height: 100%;">

</div>

---
transition: fade-out
---

# Key Challenge 1
**How to determine whether the exponential growth of cwnd is expected to continue in the next round?**

<div v-click>

→ implement the approach from HyStart

</div>

<div v-click>

- HyStart
  - Upon receiving an ACK, the amount of time elapsed from the start of the current round does not exceed half of the minRTT
  - The minimum observed RTT in the current round ($m o R T T_i$) must not be greater than 1.125 $*$ minRTT.

</div>

<div v-click>

- Condition 1 for SUSS 
  - $\Delta t_{i+1}^{a t} \le {m i n R T T}/{2}$
  - need to estimate $\Delta t_{i+1}^{a t}$

</div>

<div v-click>

- Condition 2 for SUSS 
  - $m o R T T_{i+1} <= 1.125 * m i n R T T$
  - need to estimate $m o R T T_{i+1}$

</div>

---
transition: fade-out
---

# Formulating Condition 1 for SUSS
$\Delta t_{i+1}^{a t} \le {m i n R T T}/{2}$

<div v-click>

- $\Delta t_{i+1}^{a t}$ is not known in round(i) → estimate it by its relationship with $\Delta t_{i}^{a t}$ 
  - $\Delta t_{i}^{a t} = \frac{c w n d_{i-1}}{B t l B w}$ 

</div>

<div v-click>

- As the same as slow-start, $2 * c w n d_{i-1}$ is sent during $\Delta t_i^{a t}$
  - $\Delta t_{i+1}^{a t} = \frac{2 * c w n d_{i-1}}{B t l B w} = 2 * \Delta t_{i}^{a t}$

</div>

<div v-click>

- **Condition 1**: $\Delta t_{i}^{a t} \le {m i n R T T}/{4}$

</div>

---
transition: fade-out
---

# Formulating Condition 2 for SUSS
$m o R T T_{i+1} \le 1.125 * m i n R T T$

<div v-click>

- $1.125 * m i n R T T$ → initial signs of queueing delay → beginning of congestion

</div>

<div v-click>

- suppose that $m i n R T T$ was last updated r rounds ago
  - the average increase in queuing delay since then is approximately $\frac{m o R T T_i - m i n R T T}{r}$
  - $m o R T T_{i+1} ≈ m o R T T_{i} + \frac{m o R T T_i - m i n R T T}{r}$

</div>

<div v-click>

- **Condition 2**: $m o R T T_{i} + \frac{m o R T T_i - m i n R T T}{r} \le 1.125 * m i n R T T$

</div>

---
transition: fade-out
---

# Key Challenge 2
**How to control additional data packet transmission to avoid bursts and packet loss?**

<div v-click>

- Combination of ACK clocking and packet pacing when $G_i > 2$
  - only ACK clocking → a burst of packets upon receiving each ACK train
  - only packet pacing → the measurement of $\Delta t_i^{a t}$ will no longer be accurate

</div>

<div v-click>

<img src="/fig6.png" style="margin: auto; max-width: 70%; max-height: 100%;">

</div>

<div v-click>

- In _clocking period_, SUSS always sends twice the amount of data acknowledged by that ACK
  - facilitate the measurement of $\Delta t_i^{a t}$
- In _pacing period_, the remaining data will be sent at a predetermined pace to mitigate burstiness
- _guard period_ → avoid interference with the clocking period in both the current and the next round

</div>

---
transition: fade-out
---

<img src="/fig20.png" style="margin: auto; max-width: 85%; max-height: 100%;">


---
transition: fade-out
---

<img src="/fig6.png" style="margin: auto; max-width: 50%; max-height: 100%;">

<img src="/fig8.png" style="margin: auto; max-width: 70%; max-height: 100%;">


---
layout: two-cols
transition: fade-out
---


<img src="/fig9.png" style="margin: auto; max-width: 85%; max-height: 100%;">

::right::

<img src="/fig10.png" style="margin: auto; max-width: 83.2%; max-height: 100%;">


---
transition: fade-out
---

# Performance Evaluation
- server deployment: Google server(BBR), Oracle server(CUBIC), both 3 servers located in different places
- client deployment: SUSS on server, different kinds of clients in NZ or Sweden

<div v-click>

### Throughput

<img src="/fig11.png" style="float: left; max-width: 45%; max-height: 100%;">

<img src="/fig12.png" style="margin: auto; max-width: 45%; max-height: 100%;">

</div>

<div v-click>

- CUBIC with SUSS exhibits a faster and smoother increase in cwnd and does not incur extra end-to-end delay
- SUSS has brought substantial improvements
  - SUSS can reach $c w n d^*$ more quickly than the traditional slow-start
  - SUSS avoids overshooting cwnd and maintains the consistent delivery rate

</div>

---
transition: fade-out
---

# Performance Evaluation
FCT (Flow Complete Time)

<div v-click>

### Small Flow
<img src="/fig13.png" style="margin: auto; max-width: 80%; max-height: 100%;">

<img src="/fig14.png" style="margin: auto; max-width: 80%; max-height: 100%;">

</div>

<div v-click>

- accelerating slow-start can yield significant gain for small flows
  - as many small flows predominantly reside within the slow-start phase

</div>

---
transition: fade-out
---

# Performance Evaluation
FCT (Flow Complete Time)

<div v-click>

### Large Flow
<img src="/fig15.png" style="margin: auto; max-width: 60%; max-height: 100%;">

</div>

<div v-click>

- SUSS enhances the performance of small-sized flows, it does not increase cwnd beyond its optimal value, nor does it impact the FCT of larger flows

</div>

---
transition: fade-out
---

# Performance Evaluation
Packet Loss

<div v-click>

<img src="/fig16.png" style="margin: auto; max-width: 70%; max-height: 100%;">

</div>

<div v-click>

- Pacing can significantly reduce packet density and thereby reduce the packet loss rate
- It will converge when the size of flow increase

</div>

---
transition: fade-out
---

# Performance Evaluation
Fairness

<div v-click>

- $F = \frac{\sum (x_i)^2}{n * \sum (x_i^2)}$, n is the number of flows and $x_i$ is the goodput of the i-th flow → $F \rightarrow 1$, fairness $\uparrow$

</div>

<div v-click>

<img src="/fig17.png" style="margin: auto; max-width: 60%; max-height: 100%;">

</div>

<div v-click>

- when a level of fairness is achieved among the four flows, the fifth flow starts downloading
- the initiation of the fifth flow results in an immediate decrease in the value of F , exhibiting a prolonged delay in approaching 1 when SUSS is off

</div>

---
transition: fade-out
---

# Performance Evaluation
Stability

- A large flow facing the initiation of multiple concurrent small flows
<div v-click>

<img src="/fig18.png" style="margin: auto; max-width: 50%; max-height: 100%;">
<img src="/fig19.png" style="margin: auto; max-width: 70%; max-height: 100%;">

</div>

<div v-click>

- SUSS improves FCT of small CUBIC flows without compromising the stability of the large flow

</div>

---
layout: center
class: text-center
---

# Thanks for Listening

[Paper](https://dl.acm.org/doi/10.1145/3651890.3672234) · [GitHub](https://github.com/SUSSdeveloper/SUSS) · [Conference video](https://www.youtube.com/watch?v=-vXLobRg5xQ)
