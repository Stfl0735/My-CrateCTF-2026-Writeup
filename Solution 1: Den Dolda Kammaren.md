# crate CTF 2026 – Den Dolda Kammaren (Web, Warmup)

In this CTF you have a website with a black background and a quote that says "Den Dolda Kammaren" (all capital letters). So you have nothing at the beginning. I had 2-3 ways in front of me to start solving this:

1. Look at the source of the page
2. Check whether there is a `robots.txt`
3. Use an enumeration tool such as ffuf or gobuster

I chose the first path because there was nothing to reach other endpoints and there was only writing (also, this is the warmup challenge).

At the end of the page source we see this:

```html
<body>
    <h1>Den Dolda Kammaren</h1>

    <div class="something-hidden-by-the-darkness" id="something-hidden-by-the-darkness">
        <div class="door" id="door"></div>
        <div class="chamber" id="secret-chamber">
            <p>Du hittade kammaren!</p>
            <button id="decode-btn" type="button">Visa flaggan</button>
            <p class="decoded" id="decoded"></p>
        </div>
    </div>
```

There is a hidden button, but instead of looking for a way to make it visible, I decided to understand the code behind it with the help of AI:

```html
<script>
        var KEY = 0x2A;
        var MUL = 31;
        var MUL_INV = 223;
        var STEP = 7;
        var OBFUSCATED = "J//3S4FKYLVHIMJkDsCBnwtv+jCrfBZMapDs8612bQCikuKvtn/zvF4THiUdM48=";
        document.getElementById("decode-btn").addEventListener("click", function () {
            var raw = atob(OBFUSCATED);
            var out = "";
            for (var i = 0; i < raw.length; i++) {
                var b = raw.charCodeAt(i);
                var c = ((b - KEY - STEP * i) % 256 + 256) % 256;
                out += String.fromCharCode((c * MUL_INV) % 256);
            }
            document.getElementById("decoded").textContent = out;
        });
</script>
```

In this code there is a hexadecimal `KEY` which equals 42 in decimal. There are two variables, `MUL` and `MUL_INV`, that have a special relationship which I will explain later. There is also a variable `STEP`, assigned the decimal value 7, which I will explain later too.

The `OBFUSCATED` variable is encoded in Base64, because the `atob()` function decodes Base64 in JS.

The `out` variable stores the output. In the for loop, the variable `b` is the ASCII value of the character at index `i` of the string `raw`.

In the cipher algorithm, each character of the plain flag at index `i` is encrypted as `cipher[i] = (plain[i] * MUL + KEY + STEP * i) % 256`. This means the ASCII value is first multiplied by the constant `MUL`, then shifted by `KEY` and by `STEP * i`. Because of the `STEP * i` term, the same ASCII value at different positions is encoded as a different value (as long as the message is shorter than 256 characters).

`((b - KEY - STEP * i) % 256 + 256) % 256` removes the position-based shift from the cipher. In JavaScript, `%` can return negative values, so adding 256 and taking `% 256` again keeps the result in the range 0-255. In Python you don't need it, because `%` always returns a non-negative result.

After `c` is calculated, we find the original character at index `i` with `(c * MUL_INV) % 256`. But before explaining this step, I must explain the relationship between `MUL` and `MUL_INV`.

In my claim, `MUL` is 31 and `MUL_INV` is the modular inverse of `MUL` modulo 256.

`31 * 223 = 6913 = 27 * 256 + 1` is our proof, and the following is our example, as you can see:

```
66 * 31 = 2046
2046 % 256 = 254
254 * 223 = 56642
56642 % 256 = 66
```

Here is the version I ran in Node.js:

```javascript
var KEY = 0x2A;
var MUL = 31;
var MUL_INV = 223;
var STEP = 7;
var OBFUSCATED = "J//3S4FKYLVHIMJkDsCBnwtv+jCrfBZMapDs8612bQCikuKvtn/zvF4THiUdM48=";
var raw = atob(OBFUSCATED);
var out = "";
for (var i = 0; i < raw.length; i++) {
    var b = raw.charCodeAt(i);
    var c = ((b - KEY - STEP * i) % 256 + 256) % 256;
    out += String.fromCharCode((c * MUL_INV) % 256);
}
console.log(out);
```

I removed the DOM-based parts with the help of AI, then ran the code in Node.js, because `atob()` is available in Node.js 16 and above.

Running the script prints the flag in the format `cratectf{...}`.

## What I learned

There can be different ways to solve a problem, but that is no excuse for skipping the fundamentals that matter, like basic JS, maths and discovering new things.
