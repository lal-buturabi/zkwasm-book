# Circuit Design

## 1 INTRODUCTION
WASM (or WebAssembly) is an open standard binary code format close to assembly. Its initialobjective is to provide an alternative to java-script with better performance in the current webecosystems. Benefiting from its platform independence, front-end flexibility (can be compiled fromthe majority of languages including C, C++, assembly script, rust, etc.), good isolated runtimeand speed that is close to native binary, its usage starts to arise in the distributed cloud and edgecomputing. Recently it has become a popular binary format for users to run customized functionson AWS Lambda, Open Yurt, AZURE, etc.


**The Problem.** To implement a ZKSNARK-backed WASM virtual machine, we need to connect theimplementation of WASM runtime with the proof system of ZKSNARK. In general, a ZKSNARKsystem is represented in arithmetic circuits with polynomial constraints. Therefore we need toabstract the full imperative logic of a WASM virtual machine systematically and rewrite it intoarithmetic circuits with constraints. Given two outputs, one is generated by emulating the WASMbytecode in WASM runtime that enforces the semantics of WASM specification, and the othersatisfies the constraints imposed on the arithmetic circuits. If the circuits we write preserve thesemantics, these two outputs must be the same. Hence the proof of the ZKSNARK derived from thecircuits also shows that the output is valid as a result of emulating the bytecode in WASM runtime.

**Organization of the document.** After a brief introduction to the basic ideas about how to connecta stateful virtual machine with ZKSNARK in Section 2, we describe the basic building block andingredients used to construct ZKWASM circuits in Section 3 and then present the circuits architecturein Section 4. After the architecture is settled, we discuss the circuits of every category of WASM instructions in Section 5. In addition, in Section 5.4 we discuss foreign instruction expansion whichprovides a way to extend the virtual machine for better performance and integration. In Section 6, we present the partition and proof batching technique to solve the long execution trace problem.

## 2 OVERVIEW
Throughout the paper, we use the notation $$a:A$$ to specify a variable of type $$A$$, $$F$$ to specify anumber field, and $$F_n$$ to specify a multi-dimensional vector with dimension $$n$$. We denote by $$A \rightarrow B$$ the function type from $$A$$ to $$B$$ and use $$\circ$$ for function composition. Moreover, we use $$G[i]$$ and $$G[j]$$ to specify the value of the cell of matrix G at the $$i$$ th row and $$j$$ th column.

### 2.1 WASM Run-Time as a State Machine
We consider the WASM virtual machine as a gigantic program, with the input as a tuple $$(I(C, H), E, IO)$$ ,where $$I$$ is a WASM executable image that contains a code image $$C$$ and an initial memory $$H$$, $$E$$ is its entry point, and IO represents the **(stdin, stdout)** firmware. In the serverless setup, the WASM run-time starts with an initial state based on the loaded image $$I$$, then jumps to the entry point $$E$$ and starts executing the bytecode based on the WASM specification.

Internally the WASM run-time maintains a state S denoted by a tuple ($$iaddr$$, $$F$$, $$M$$, $$G$$, $$Sp$$, $$I$$, $$IO$$) where $$iaddr$$ is the current instruction address, $$F$$ is the calling frame with a `depth` field, $$M$$ is thememory state, $$Sp$$ is the stack and $$G$$ is the set of global variables. The run-time simulates thesemantic of each instruction start at $$E$$ until it reaches the exit. The instructions it simulates form an execution trace $$[t_0, t_1, t_2, t_3, \cdots]$$ and each transition $$t_i$$ is a function between states that takes an input $$s: S$$ and outputs a new state $$s': S$$.

For simplicity, we will use the notation of record field to specify a field in state $$𝑠:S$$. For example, $$s.iaddr$$ denotes the current instruction address of state 𝑠, $$𝑠.IO.stdin$$ denotes the input of state 𝑠, etc. We also use $$s.iaddr.op$$ to denote the opcode (operationcode that specifies the operation to be performed) at address $$s.iaddr$$ in the code section $$C$$ of image $$I$$.

Based on the above definition, we define the criteria for a list of state transitions to be validunder $$(I(C, H), E, IO)$$, as follows.

-- **Definition 2.1** (Valid Execution Trace). Given a WASM machine with input $$(I(C, H), E, IO)$$, and $$s_0$$ is the initial state with $$s_0.𝑖𝑎𝑑𝑑𝑟 = E$$. A valid execution trace is a list of transition functions 𝑡𝑖 suchthat the following holds:
(1) Each $$t_0$$ matches the semantic of the instruction $$op$$ at the entry $$iaddr = E$$.
(2) For all $$𝑘$$, $$s_k = t_{k-1} \circ \cdots \circ t_1 \circ t_0 (s_0)$$, $$t_k$$ enforces the semantics of $$s_k.iaddr.op$$.
(3) If $$s_e$$ is the last state, then the depth of the calling frame is zero: $$se.F.depth = 0$$.

We take the output of the final state $$s_e.IO.ouptut$$ to be the result of the WASM run-time withinput $$(I(C, H), E, IO)$$. The $$output$$ is a valid result if and only if there exists an valid execution sequence $$[t_0, t_1, \cdots]$$ such that $$s_e$$ is the last state of $$t_i$$ under $$(I(C, H), E, IO)$$.

## 2.2 Succinct Proof of a Program
Compared with a standard WASM run-time, ZKWASM aims to provide a proof to prove that theoutput is valid so that it can be used in scenarios which require trustless and privacy computation.Moreover, the verifying algorithm needs to be simple in the sense of complexity to be useful inpractical. Before we dive into how to construct such a proving and verifying scheme for the complex WASM run-time, we go through a few basics about how to construct such a scheme for functions.

Suppose that we have a pure function $$f$$, a list of parameters $$params$$ for $$f$$ , an entity $$A$$ that calculates $$r=f(params)$$ and an entity $$B$$ that would like to know $$r$$ but does not willing to do the exact computation. A scheme that enables $$A$$ to prove to $$B$$ about the correctness of $$r$$ is of great interests in cryptography design if the complexity for $$B$$ to verify the proof is negligible comparing.

Topics about constructing ZKSNARK has been well studied in the literature where functions $$f$$ are defined by a computational program $$P$$. A common approach for constructing such ZKSNARK is to turn the program $$P:F_m \rightarrow F_r$$ into a special form of constraint systems $$C_i(x) = 0$$ (where $$C: F_n \rightarrow F$$), such that for any parameters $$param: F_m$$ of $$P$$, there exists a unique vector of witness $$w: F_{n+𝑚+r}$$ and a unique vector of result $$r: F_r$$ that satisfy

$$$C_i(param_0, param_1,\cdots, w_0, w_1,\cdots,r_0, r_1,\cdots) = 0$$$

We call such constraint system arithmetic circuits (see 2.3 for the a precise definition). Once the arithmetic circuits are constructed based on $$P$$, the problem of proving $$P(params) = r$$ becomes the problems of finding witness vector $$w$$ and prove that the vector $$v = (params;w;r)$$ satisfies $$C(v) = 0$$.

Once the problem of constructing ZKSNARK for a program $$P$$ is turned into the problem of constructing ZKSNARK for the correspondent constraint system $$C$$, various approach can be applied based on the shapes of $$C$$. The basic idea to construct ZKSNARK for $$C$$ is to turn the proof for the constraint system $$C$$ into proofs of polynomial evaluations.

## 2.3 Arithmetic Circuits
Arithmetic circuits are a crucial building block in the ZKSNARK of a program. Among various arithmetic circuit systems investigated recently, we use the Halo2’s circuit system for its flexibility in customization and better integration with polynomial lookup and shuffle to representing set inclusion.

A circuit is defined by a tuple $$(G, C)$$ where $$G$$ is a $$n$$ column matrix with rows to befilled later and $$C$$ is a set of constraints on a row basis. More precisely, suppose that each cell in $$G$$ is indexed (relative to a row $$l$$) as $$G_{l,c,r} = G[l+r][c]$$, then each constraint $$C_i$$ in $$C$$ is one of thefollowing form

**Equation 1.**
$$$
    C_i(l) = \begin{cases} P \left(G_{l, c_0, r_0}, G_{l, c_1, r_1}, \cdots, G_{l, c_k, r_k}\right) = 0 \\
        E \left(G_{l, c_0, r_0}, G_{l, c_1, r_1}, \cdots, G_{l, c_k, r_k}\right) \in T
    \end{cases}
$$$
where $$c_k$$, $$r_k$$ are constants, $$P$$, $$E$$ and $$T$$ are fixed multi-linear polynomials.

In the rest of this document, we use $$cur$$ as the current row that $$C_i$$ is apply on and use the abbrevation $$G_{r,c}$$ to denote $$G[cur+r][c]$$ to emphasize the column $$c$$. With this notation, we see that the constraint system $$C$$ provides a flexible way for us to define constraint of cells of a row and their neibours.

For example, given the following summarize function $$sum$$
```
function sum(v) {
    for (acc=0,i=0;i<v.length;i++) {
        acc +=v[i];
    }
    return acc;
}
```
The circuit for it can be constructed as in **Table 1** and the constraint system enforced on each row is defined in **Equation 2**.

**Table 1.** Circuit matrix of sum
|  s  |       acc        | operand |
| --- | ---------------- |-------  |
|  1  |     $$acc_0 = 0$$    |   $$v_0$$    |
|  1  | $$acc_1 = acc_0 +v_0$$ |   $$v_1$$    |
|  1  |       ...        |   ...   |
|  0  |      $$acc_k$$     |   nil   |

**Equation 2.** Constraint of sum
$$$
 C(cur) = \begin{cases}
     s.(cur) \times (acc.(cur) + operand.(cur) - acc.(cur+1)) = 0 \\
     s.(cur) \times (1-s.(cur)) = 0
 \end{cases}
$$$

**Remark:** Notice that the first constraint ensures that addition is applied to each row except forthe last row, and the second constraint enforces that $$s$$ is either 1 or 0).

Motivated by the above example, we present the formal definition of arithmetic circuits as follows.

**Definition 2.2** (Arithmetic Circuit). An arithmetic circuit is a matrix $$G$$ with $$m$$ columns equipped with a constraint system $$C$$ that each constraint $$C_i$$ of $$C$$ is defined as in **Equation 1** and for each row $$cur$$ in the matrix $$C(cur)$$ holds.
