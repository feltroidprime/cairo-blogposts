## Defeating Malicious Provers in Cairo with Developer Schizophrenia. How to have a Zero Knwoledge Proof mentality in practice. 

How to use the power of Cairo hints by being two developers at the same time. 

Hints in Cairo can at the same time be very powerful when utilized properly, and very dangerous if malicious actions are not prevented. In this post, we'll review through two examples how to defeat malicious provers by forcing them to comply to provide you what you want. 

https://www.cairo-lang.org/docs/how_cairo_works/hints.html


### 1. A brief recap about Cairo, provers and verifiers. 

Cairo is a turing-complete programming language that you can prove. It is not like your traditional programming language when you run the code on your machine or a cloud server and that's it. Here, there is two people involved: 
- The prover who runs the full .cairo code and outputs a proof a computation. 
- the verifier who validates the proof related to the .cairo file, without running the .cairo file. 

The scaling property of STARK proofs is such that validating a proof is cheaper than running the original .cairo in the first place. 

Note that the proof is only related to Cairo code inside the .cairo file. This important because when you're writing .cairo files, you can write two types of code. Hints, that are only executed by the prover but it is not proven, and the cairo code that is proven. 

The idea is that you can create entries for some values in you cairo code and ask inside the hint to the prover to fill those values. 

```
// Asserts that value is in the range [lower, upper).
// Or more precisely:
// (0 <= value - lower < RANGE_CHECK_BOUND) and (0 <= upper - 1 - value < RANGE_CHECK_BOUND).
//
// Prover assumption: 0 <= upper - lower <= RANGE_CHECK_BOUND.
func assert_in_range{range_check_ptr}(value, lower, upper) {
    assert_le(lower, value);
    assert_le(value, upper - 1);
    return ();
}

// Returns the floor value of the square root of the given value.
// Assumptions: 0 <= value < 2**250.
@known_ap_change
func sqrt{range_check_ptr}(value) -> felt {
    alloc_locals;
    local root: felt;

    %{
        from starkware.python.math_utils import isqrt
        value = ids.value % PRIME
        assert value < 2 ** 250, f"value={value} is outside of the range [0, 2**250)."
        assert 2 ** 250 < PRIME
        ids.root = isqrt(value)
    %}

    assert_nn_le(root, 2 ** 125 - 1);
    tempvar root_plus_one = root + 1;
    assert_in_range(value, root * root, root_plus_one * root_plus_one);

    return root;
}


```

### 2. A simple yet detailed example

Here is an example from a small library I wrote for packing felts at the bit level. 
In Cairo, every cell in the memory is a field element, meaning that the base type of the Cairo languauge is a field element. Precisely, they're an integer between 0 and P.
This is unlike the C language for example, when you have raw access to every bit in the memory.


```
from starkware.cairo.common.math_cmp import is_le
from starkware.cairo.common.cairo_builtins import BitwiseBuiltin

func get_felt_bitlength{range_check_ptr, bitwise_ptr: BitwiseBuiltin*}(x: felt) -> felt {
    alloc_locals;
    local bit_length;
    %{
        x = ids.x
        ids.bit_length = x.bit_length()
    %}

    let le = is_le(bit_length, 252);
    assert le = 1;
    assert bitwise_ptr[0].x = x;
    let (n) = pow(2, bit_length);
    assert bitwise_ptr[0].y = n - 1;
    tempvar word = bitwise_ptr[0].x_and_y;
    assert word = x;

    assert bitwise_ptr[1].x = x;

    let (n) = pow(2, bit_length - 1);

    assert bitwise_ptr[1].y = n - 1;
    tempvar word = bitwise_ptr[1].x_and_y;
    assert word = x - n;

    let bitwise_ptr = bitwise_ptr + 2 * BitwiseBuiltin.SIZE;
    return bit_length;
}


```
### 3. A more complex example requiring deeper mathematics 

Let's say you need to compute a square root of. 

```
func get_square_root_mod_p{range_check_ptr}(x: Uint256, G:Uint256, P:Uint256) -> (success: felt, res: Uint256) {
    alloc_locals;

    let is_zero = u255.eq(x, Uint256(0, 0));
    if (is_zero == 1) {
        return (1, Uint256(0, 0));
    }


    local sqrt_x: Uint256;
    local sqrt_gx: Uint256;

    %{
        HINT CONTENT REMOVED FOR EDUCATIONAL/READABILITY PURPOSES

        DUE TO THE LOCAL VARIABLES DEFINED ABOVE, THE PROVER CAN WRITE ANY ARBITRARY FELT VALUES INSIDE sqrt_x OR sqrt_gx, AND STILL CREATE A VALID STARK PROOF OF EXECUTION.

        THIS HINT ONLY [[[SUGGESTS]]] THE PROVER TO COMPUTE AND RETURN THE SQUARE ROOT OF [X MODULO P] IF IT EXISTS, OTHERWISE COMPUTE THE SQUARE ROOT OF [X*G MODULO P] AND RETURN IT.

    %}

    // Verify that the values computed in the hint are what they are supposed to be
    %{ print_u_256_info(ids.sqrt_x, 'root') %}
    let gx: Uint256 = mul(generator, x);
    if (success_x == 1) {
        let sqrt_x_squared: Uint256 = mul(sqrt_x, sqrt_x);

        // Note these checks may fail if the input x does not satisfy 0<= x < p
        // TODO: Create a equality function within field_arithmetic to avoid overflow bugs
        let check_x = u255.eq(x, sqrt_x_squared);
        assert check_x = 1;
    } else {
        // In this case success_gx = 1
        let sqrt_gx_squared: Uint256 = mul(sqrt_gx, sqrt_gx);
        let check_gx = u255.eq(gx, sqrt_gx_squared);
        assert check_gx = 1;
    }

    // Return the appropriate values
    if (success_x == 0) {
        // No square roots were found
        // Note that Uint256(0, 0) is not a square root here, but something needs to be returned
        return (0, Uint256(0, 0));
    } else {
        return (1, sqrt_x);
    }
}

```

### 4. 




### 4. Summary : when and how to use the power of hints in Cairo 

In some way
Hints are very useful when the verification cost of computing some value is cheaper than finding the value. 
Some other examples can include finding solutions to some equations.
What 
What will be the properties of the inputs you're going to provide ?
What inequality defines them ? 
What would happen if the input is 0 ? 1 ? Do you need to prevent those cases ? 

Always think about the lower and higher range of