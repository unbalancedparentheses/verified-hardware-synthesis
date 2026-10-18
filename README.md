# Verified Hardware Synthesis

**You write the spec. An e-graph generates every correct way to build it. AI helps explore that space,
synthesis tells you which variant is best, and Lean guarantees that whatever comes out is correct.**

Status: idea and research plan. No code yet.

## The problem

Formal verification of hardware works, but it is expensive. Proofs are usually written by hand,
after the design, and they cost more than the design itself. Sparkle's RV32 core, for example,
ships with 102 hand-written theorems. So almost nobody verifies hardware formally, and the
proofs that do exist usually stop at a model: the step from that model to Verilog, and from
Verilog to a gate netlist, is trusted.

Commercial high-level synthesis (HLS) tools already explore many implementations of an algorithm
automatically, but without proofs. Theorem-prover-based hardware languages give proofs, but
without automatic exploration. Research has started applying e-graphs to datapath optimization
(see ROVER under Related work), but no tool combines automatic exploration with machine-checked
proofs that reach the netlist.

## The idea

Combine four pieces, each with a clear role:

| Piece | Role | Trusted? |
|---|---|---|
| **Spec** | What the circuit computes (e.g. 4×4 int8 matrix multiply, 32-bit accumulation), plus constraints: area, frequency, latency | It *is* the definition of correct |
| **Rewrite rules** | Meaning-preserving transformations, each proved once in Lean and reused forever | Yes, proved |
| **E-graph** | Applies the rules and stores, compactly, *all* equivalent versions of the design | Yes, a verified engine |
| **Extraction** | Picks the best variant: a cheap cost model filters, real synthesis (Yosys / OpenROAD) measures the finalists | Not needed for correctness |
| **AI** | Guides the search, proposes new rules, proposes whole designs, writes proofs | **No.** Untrusted by design |

**The AI proposes, Lean decides.**

- A rule the AI invents is only admitted once it is proved in Lean.
- A design the AI proposes outside the rule space is only accepted if its equivalence with the
  spec is proved or checked.
- If the AI is wrong, the worst case is a slower or bigger circuit, never an incorrect one.

### A tiny example

Spec: `y = a*b + a*c`

- Variant 1: `a*b + a*c`. Two multipliers and one adder: faster, more area.
- Variant 2: `a*(b+c)`. One adder and one multiplier: less area, slightly longer critical path.

Both live in the e-graph and both are proved equal to the spec. Extraction picks one depending on
whether the constraint is area or speed. Scale this to a multiply-accumulate array with hundreds
of operations and the space of correct variants becomes far too large to explore by hand.

### Why AI is needed at all

E-graphs explode if every rule is applied everywhere. Equality saturation on real datapaths
needs guidance: which rewrites to try, where, and when to stop. That is a search problem where
learned or LLM-based heuristics help a lot, and where mistakes cost nothing but time, because
every candidate is checked.

## Where a 1000x improvement could come from

A nicer HDL, more IP blocks or faster simulation would be incremental. The large gains are on
other axes:

1. **Proof cost.** You write the spec; the system produces implementation *and* proof. The metric
   is human hours per verified gate, which today is weeks of expert work.
2. **Spec to design.** Instead of hand-designing a microarchitecture and then proving it, you
   explore thousands of variants that are correct by construction.
3. **Trust down to the netlist.** Every step, Lean → Verilog → gate netlist, is either proved or
   checked per design (translation validation), with an explicit report of what is proved,
   tested and still trusted.
4. **Existing designs.** Import existing Verilog/SystemVerilog and verify it, not only new code
   written in a new language. The market is the RTL that already exists.

## What would make it useful in practice

A clever research prototype is not enough. Chip and FPGA teams will only use it if it fits how they
already work and beats what they already have.

1. **No new language required.** Input is what teams already have: a Verilog/SystemVerilog
   datapath, or the C/C++ golden reference model that most hardware teams maintain anyway. Output
   is a drop-in Verilog module with the same interface. Writing specs in Lean should be optional.
2. **Wins that survive real synthesis.** Commercial synthesis tools (Design Compiler, Genus) and
   Yosys already do some rewriting. A variant is only valuable if it is still smaller or faster
   *after* their optimizations. Every benchmark must be measured end to end, post-synthesis.
3. **Certificates checked by tools they already trust.** Besides the Lean proof, emit an
   equivalence problem that the customer's own formal equivalence checker (Formality, Conformal,
   SymbiYosys) can confirm. Nobody has to trust Lean, or us, to adopt it.
4. **Numerical precision with proved error bounds.** Many of the biggest savings come from
   narrower widths and cheaper arithmetic: fixed-point instead of floating-point, truncated
   multipliers, approximate accumulation. If a rewrite may change results, it is only allowed
   with a proved error bound (Herbie-style, but for hardware). That turns "correct" into
   "correct within ε", where much of the area is saved.
5. **Fast enough for CI.** Minutes, not days, on a typical block, so it runs on every change.
6. **Explainable results.** Show the designer which rewrites produced the chosen variant and the
   area/timing trade-off curve, not just a black-box netlist.
7. **A sharp first wedge.** Datapath-heavy blocks where area and power matter and verification is
   expensive: ML accelerator MAC arrays, DSP filters, video and image pipelines, ECC/CRC units.
   FPGA teams iterate fastest and are the easiest first users; ML-accelerator startups feel the
   area and power pain most.

## Scope and limits

E-graphs are very good at **combinational logic and dataflow**: arithmetic, datapaths, adder and
multiplier trees, resource sharing.

**Sequential logic** (pipelining, scheduling, retiming) is harder. There, rewrites must change
*when* things happen, not just *what* is computed. Research exists but is much less mature. This
project starts with arithmetic datapaths and treats sequential logic as later work.

The proofs are only as good as the spec. A circuit that satisfies a wrong or vacuous theorem is
still wrong. Passing Lean's type checker is not a correctness guarantee on its own.

## Building blocks

- **[Sparkle](https://github.com/Verilean/sparkle)**: a hardware description language in Lean 4
  (inspired by Clash). You write a Lean spec, prove properties, implement in a `Signal` DSL, and
  generate SystemVerilog. It includes a Linux-booting RV32IMA core with 102 proofs, a BitNet
  accelerator with 60+ theorems, and verified arbiters and AXI4-Lite. It is young: v0.1.0 in
  January 2026, essentially one main author. Its Lean → Verilog step is trusted, not verified.
  The plan is to target its intermediate representation and contribute upstream, **not to fork
  it**.
- **[truth_research](https://github.com/lambdaclass/truth_research)**: a Lean-verified
  equality-saturation engine. This is the verified e-graph core.
- **verified-search**: experiments combining LLMs, e-graphs and formal verification.
- **[Concrete](https://github.com/unbalancedparentheses/concrete2)**: the same philosophy applied to
  software. A systems language whose compiler is written in Lean, with explicit claims about what
  is proved, tested and trusted.
- **Yosys, OpenROAD, SymbiYosys**: open synthesis, place-and-route and formal equivalence
  checking, used for cost measurement and netlist-level checks.

## First experiment (about two months)

Goal: show measurable area/timing wins with zero loss of correctness on one real datapath.

1. **Spec**: a 4×4 int8 multiply-accumulate (MAC) array with 32-bit accumulators, written in Lean.
2. **Rules**: a small library of proved rewrites (associativity and commutativity of adders,
   distributivity, multiplier sharing, adder-tree reshaping, constant folding, width narrowing
   backed by overflow proofs).
3. **Exploration**: run the verified e-graph, with and without AI guidance, and extract hundreds of
   variants.
4. **Measurement**: synthesize every variant with Yosys and OpenROAD on an open process (sky130),
   and plot the area/timing curve against a hand-written baseline.
5. **Netlist check**: equivalence-check the emitted Verilog against the synthesized netlist for
   the chosen variants.

If it works: a credible demo for the bigger vision, and a conversation with chip companies.
If it does not: little is lost, and the e-graph work feeds back into `truth_research`.

## Later

- Translation validation of Sparkle-emitted Verilog, then a formal semantics for the Verilog
  subset it emits.
- AI-generated proofs for sequential parts: control logic, FSMs, pipelines.
- Importing existing RTL and verifying it.
- Larger targets: verified accelerator generators (parametric N×N), RISC-V blocks, verified IP
  that others integrate.
- Possibly a small chip on [Tiny Tapeout](https://tinytapeout.com/) as a public demo.

## Risks

- **Market.** Chip design tools are dominated by Synopsys, Cadence and Siemens, and sales to chip
  companies are slow.
- **Dependency.** Sparkle is a young project with one main author. The mitigation is to contribute
  upstream and work with its author.
- **Toolchain churn.** Lean releases monthly. Sparkle's README already notes `bv_decide` problems
  on Lean 4.28.
- **E-graph blow-up.** On large designs the space may be too big even with guidance. Starting with
  small datapaths tests this early.
- **Time horizon.** The full vision is multi-year.

## Open questions

- Which cost model inside the e-graph correlates well enough with real post-synthesis area and
  timing?
- How much does AI guidance actually help compared with classic scheduling heuristics for
  equality saturation?
- What is the right intermediate representation: Sparkle's IR, a dedicated dataflow IR, or both?
- Is it better to prove the Verilog emitter once, or validate each emitted design?
- Who pays first: chip companies, safety-critical FPGA users, or verified-IP buyers?

## Related work

- [CktFormalizer](https://arxiv.org/pdf/2605.07782): autoformalization of natural language into
  circuits, built on Sparkle. Found designs that passed Lean checks but failed simulation.
- [CircuitProver](https://arxiv.org/pdf/2607.27259): agentic Lean 4 theorem proving for hardware
  (Chisel → Lean).
- [Rtl2lean](https://arxiv.org/pdf/2607.16855): automated SystemVerilog → Lean translation.
- Kami and Kôika (Coq, MIT): verified hardware design in a proof assistant.
- egg / equality saturation: the e-graph technique this project relies on.
- ROVER (Coward, Constantinides et al., Imperial College): e-graph-based datapath optimization for
  hardware, without machine-checked proofs.
- Herbie: rewriting floating-point expressions for accuracy; the model for proved error-bounded
  rewrites.
