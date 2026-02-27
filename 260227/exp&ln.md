# The summary of method to calculate the Exp and Ln
This section bridges the gap between fixed-point theory and actual hardware logic. Below is the technical breakdown of the mathematical principles behind the `exp(x)` and `ln(x)` modules, translated and formatted for your design documentation.
This is a solid breakdown of the fixed-point logic required for hardware acceleration. Let’s translate and refine this into a professional design guide, optimized for clarity and technical accuracy.

---

## Hardware Principles: `exp(x)` and `ln(x)` Implementation

### 1. The Logic Behind `exp(x)`

Based on the `exp_module` logic, the hardware implementation follows three primary mathematical steps to avoid complex power-series calculations:

#### Step 1: Base Conversion

Calculating $2^z$ is significantly cheaper in hardware than $e^x$ because powers of $2$ translate into simple bit-shifting operations.

* **Math:** $e^x = 2^{x \cdot \log_2(e)}$
* **Hardware Realization:** `x_log2e_S10Q14 = x_S9Q10 * 6'sd23;`. Here, `23` is the fixed-point representation of $\log_2(e) \approx 1.4427$ (specifically, $1.4427 \times 2^4 \approx 23$).

#### Step 2: Splitting Integer and Fractional Parts

Let $z = x \cdot \log_2(e)$. We decompose $z$ into an integer part $I$ and a fractional part $F$ ($z = I + F$).

* **Math:** $2^z = 2^{I + F} = 2^I \cdot 2^F$
* **Hardware Realization:** Use bit-slicing to extract `x_int` (the integer) and `x_decimal` (the fraction).

#### Step 3: Computation and Merging

* **Calculating $2^I$:** This is implemented as a **shifter**. For negative exponents, the code uses `12'h800 >> x_int_safe` to simulate $2^{-I}$.
* **Calculating $2^F$:** Since $F$ is constrained to $[0, 1)$, we use a **linear approximation** (a first-order Taylor expansion). Your code implements a simplified `1 - 0.5*F` logic: `temp_1Q14 = 15'h4000 - {2'b0, x_decimal_0Q14[13:1]}`.
* **Merging:** The final result is obtained by multiplying the two parts: `temp_y = temp_2_int * temp_1Q14`.

---

### 2. The Logic Behind `ln(x)`

In high-performance hardware, `ln(x)` is typically implemented using the **Normalization and LUT (Lookup Table)** method.

#### Step 1: Normalization

Leveraging binary representation, any positive number $x$ can be expressed in scientific notation: $x = M \cdot 2^E$.

* **$M$ (Mantissa):** The significand, normalized to the range $[1, 2)$.
* **$E$ (Exponent):** The power of 2, determined by **Leading Zero Detection (LZD)** logic.

#### Step 2: Logarithmic Decomposition

* **Math:** $\ln(x) = \ln(M \cdot 2^E) = \ln(M) + \ln(2^E) = \ln(M) + E \cdot \ln(2)$

#### Step 3: Component Calculation

1. **Calculate $E \cdot \ln(2)$:** $E$ is an integer obtained from the LZD logic, and $\ln(2)$ is a constant ($\approx 0.693$). This is a simple multiplication.
2. **Calculate $\ln(M)$ via LUT:** Since $M$ is strictly within $[1, 2)$, we can pre-calculate $\ln(1.0)$ through $\ln(2.0)$ and store them in a ROM. The fractional bits of $M$ serve as the address for the **Lookup Table (LUT)**.

#### Step 4: Final Summation

The hardware adds the LUT result ($\ln(M)$) to the product ($E \cdot \ln(2)$) to produce the final fixed-point $\ln(x)$ value.

---

### Summary Comparison

| Function | Hardware "Secret Sauce" | Mathematical Transformation |
| --- | --- | --- |
| **`exp(x)`** | **Shifting + Linear Approximation** | Converts $e^x \rightarrow 2^z$; uses shifters for integer powers. |
| **`ln(x)`** | **Leading Zero Detection + LUT** | Extracts binary exponent $E$; scales input to $[1, 2)$ for table lookup. |

---

## Technical Design Guide: Fixed-Point `exp(x)` and `ln(x)`

### 1. The `exp(x)` Workflow

In LLM contexts (like Softmax), we typically process $x \le 0$ (after the $x - \text{max}(x)$ normalization).

* **Input ($x$):**
* **Polarity:** Since inputs are negative (e.g., $-1.0, -2.5, -10.0$), the type **must** be `signed`.
* **Format Recommendation:** Use a signed fixed-point format (e.g., `S9Q10`), where the MSB is the sign bit.


* **Output ($y = e^x$):**
* **Polarity:** By definition, $e^x > 0$. Therefore, the output does not require a sign bit and should be defined as `unsigned`.
* **The Bit-Width Trap:** If you use a $Q25$ format, representing $e^0 = 1.0$ requires at least **26 bits** (1 integer bit + 25 fractional bits). Otherwise, the value $2^{25}$ will overflow or be misinterpreted as a negative number if treated as signed.



---

### 2. The `ln(x)` Workflow

Assuming a scenario where the input $x$ is always greater than $1$:

* **Input ($x$):**
* **Polarity:** Since $x > 1$, the input is always positive. You can use `unsigned`.
* **Bit-Width:** Because $x > 1$, you must allocate sufficient integer bits. For example, `U8Q16` allows values up to $255$.


* **Output ($y = \ln(x)$):**
* **Polarity:** Since $\ln(x) > 0$ for all $x > 1$, the output can be `unsigned`.
* **Caution:** If your input $x$ can fall within the $(0, 1)$ interval, $\ln(x)$ becomes negative, and the output **must** be changed to `signed`.



---

### 3. General Design Principles Summary

| Function | Input Range | Input Type | Output Range | Output Type | Key Consideration |
| --- | --- | --- | --- | --- | --- |
| **`exp(x)`** | $x \le 0$ | **`signed`** | $(0, 1]$ | **`unsigned`** | Result $1.0$ needs 1 integer bit to prevent overflow. |
| **`ln(x)`** | $x > 1$ | **`unsigned`** | $[0, +\infty)$ | **`unsigned`** | If input includes $(0, 1)$, output must be **`signed`**. |
| **`tanh(x)`** | $(-\infty, +\infty)$ | **`signed`** | $(-1, 1)$ | **`signed`** | Symmetric functions must be signed on both ends. |

---

### 4. Why "Signed" Awareness is Critical

As seen in your Testbench error ($real\_y = \$itor(y) / 2^{25}$), the simulator misinterpreting the MSB as a sign bit ruins the floating-point conversion.

* **In FPGA Logic:** The `signed` keyword dictates whether the hardware uses **Arithmetic Right Shifts** (`>>>`) and whether multipliers perform **sign extension**.
* **In Simulation:** `signed` determines whether your fixed-point hex value is converted to a positive or negative `real` number.

---

## The "Four-Step" Design Methodology

### Step 1: Physical Range Analysis

Determine the mathematical boundaries before touching binary code.

* **For `exp(x)` ($x \le 0$):** Input $x \in [-10, 0]$, Output $y \in (0, 1]$.
* **For `ln(x)` ($x \ge 1$):** Input $x \in [1, 256]$, Output $y \in [0, 5.54]$ (since $\ln(256) \approx 5.54$).

### Step 2: Choose Quantization Factor ($Q$)

Select $2^Q$ based on precision requirements.

* **Formula:** $\text{Fixed-Point Value} = \text{round}(\text{Real Value} \times 2^Q)$
* **Example:** For $Q=10$ (factor of $1024$), an input of $-1.0$ becomes $-1024$.

### Step 3: Bit-Width Allocation

1. **Integer Bits ($I$):** $I = \lceil \log_2(\text{Max\_Abs\_Value}) \rceil$.
2. **Sign Bit ($S$):** $S=1$ if negative values exist; else $S=0$.
3. **Total Width ($W$):** $W = S + I + Q$.

> **Example: `exp(x)` input $x \in [-10, 0]$ with $Q=10$**
> * Max absolute value is $10$.
> * $\log_2(10) \approx 3.32 \rightarrow I = 4$.
> * Needs sign bit $\rightarrow S = 1$.
> * **Total:** $1 + 4 + 10 = 15$ bits ($S5Q10$).
> 
> 

### Step 4: Verification & Overflow Check

Check your boundaries in Hex/Binary.

* **Max Positive Check:** For $U1Q25$ (26 bits total), $1.0$ is $2^{25} = 33,554,432$ (`26'h200_0000`). If your register is only 25 bits, this overflows to $0$.
* **Negative Check (Two's Complement):** $-10$ in $S5Q10$ is $-10240$.
* $10240 \rightarrow$ `15'b010_1000_0000_0000`
* Invert and add one $\rightarrow$ `15'b101_1000_0000_0000`.