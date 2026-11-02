# Placement Study Notes — Part 7: Aptitude (Worked Problems and Practice Set)

*Extends the formula sheet in Part 6, J11. Every example is solved step by step; the practice set at the end has answers and short reasons. Aim to solve each question in 45 to 90 seconds.*

## 0. How to use these notes and how to approach aptitude tests

- **Learn the method, not the answer:** read the formula, cover the solution, solve, compare.
- **Time plan:** typical sections give about 1 minute per quantitative question. Skip a question after 90 seconds and come back; unattempted questions rarely hurt more than wrong ones when there is negative marking, so check the marking scheme first.
- **Techniques that save time:** estimate before calculating, eliminate impossible options (units digit, parity, magnitude), convert percentages to fractions, use the **LCM method** for work problems, use ratios instead of absolute values, and write units on every line.
- **Daily routine:** 20 to 30 questions per day mixed across topics, one timed 30-minute sectional per week, and an error log (topic, what went wrong, shortcut missed).

**Contents:** A1 Mental maths and number sense · A2 Number system · A3 Percentages · A4 Profit, loss, discount · A5 Simple and compound interest · A6 Ratio, mixtures, ages · A7 Averages · A8 Time, speed, distance (trains, boats) · A9 Time and work, pipes · A10 Permutations and combinations · A11 Probability · A12 Series and sequences · A13 Algebra and mensuration · A14 Data interpretation · A15 Logical reasoning · A16 Verbal ability · A17 Practice set (30 questions with answers).

---

## A1. Mental maths and number sense

**Fraction to percentage table (memorize):** 1/2 = 50%, 1/3 = 33.33%, 2/3 = 66.67%, 1/4 = 25%, 3/4 = 75%, 1/5 = 20%, 1/6 = 16.67%, 1/7 ≈ 14.28%, 1/8 = 12.5%, 3/8 = 37.5%, 1/9 = 11.11%, 1/10 = 10%, 1/11 ≈ 9.09%, 1/12 ≈ 8.33%, 1/15 ≈ 6.67%, 1/16 = 6.25%, 1/20 = 5%.

**Squares to know:** 11² = 121, 12² = 144, 13² = 169, 14² = 196, 15² = 225, 16² = 256, 17² = 289, 18² = 324, 19² = 361, 20² = 400, 25² = 625. **Cubes:** 2³ = 8, 3³ = 27, 4³ = 64, 5³ = 125, 6³ = 216, 7³ = 343, 8³ = 512, 9³ = 729, 10³ = 1000.

**Fast multiplication tricks**

| Trick | Example |
| --- | --- |
| x 5 = divide by 2, then x 10 | 48 x 5 = 24 x 10 = 240 |
| x 25 = divide by 4, then x 100 | 36 x 25 = 9 x 100 = 900 |
| x 9 = x 10 minus the number | 47 x 9 = 470 - 47 = 423 |
| x 11 (two digits ab) = a, (a + b), b | 23 x 11 = 2 (2+3) 3 = 253 |
| squares ending in 5: n5² = n(n+1) then 25 | 35² = 3 x 4 = 12, then 25 = 1225 |
| (a + b)(a - b) = a² - b² | 52 x 48 = 50² - 2² = 2496 |
| near 100: use deviations | 97 x 94: (100-3)(100-6) = 100 - 9 + 18 = 9118 hmm: 100x(100 - 3 - 6) + 3x6 = 9100 + 18 = 9118 |
| percent swap: x% of y = y% of x | 8% of 50 = 50% of 8 = 4 |

**Divisibility rules:** 2 (even last digit), 3 (digit sum divisible by 3), 4 (last two digits), 5 (ends 0 or 5), 6 (2 and 3), 8 (last three digits), 9 (digit sum divisible by 9), 11 (difference of sums of alternate digits is 0 or a multiple of 11), 7 (double the last digit and subtract from the rest, repeat).

**Approximation tips:** round to convenient numbers (49.8% of 402 ≈ 50% of 400 = 200), compare answer options first, use fractions for percentages (12.5% = 1/8).

---

## A2. Number system

**Key ideas:** primes, factors, HCF (GCD), LCM, remainders, unit digits, factorial zeros. `HCF x LCM = product of the two numbers` (only for two numbers). HCF is the product of **lowest** powers of common primes; LCM is the product of **highest** powers of all primes.

**Example 1 (HCF and LCM).** Find HCF and LCM of 12, 18, 24. `12 = 2² x 3`, `18 = 2 x 3²`, `24 = 2³ x 3`. HCF = 2 x 3 = **6**; LCM = 2³ x 3² = **72**.

**Example 2 (number of factors).** For `n = p^a q^b r^c`, count = (a + 1)(b + 1)(c + 1). `360 = 2³ x 3² x 5` → 4 x 3 x 2 = **24 factors**.

**Example 3 (remainders by cycles).** Remainder of 2^100 divided by 7: `2³ = 8 ≡ 1 (mod 7)`, and `100 = 3 x 33 + 1`, so `2^100 ≡ (2³)^33 x 2 ≡ 2`. **Remainder 2.**

**Example 4 (unit digit).** Cycles of unit digits: 2 → 2,4,8,6; 3 → 3,9,7,1; 7 → 7,9,3,1; 4 → 4,6; 9 → 9,1; 0,1,5,6 stay fixed. Unit digit of 7^123: cycle length 4, `123 mod 4 = 3` → third element = **3**.

**Example 5 (trailing zeros of n!).** Count factors of 5: `floor(n/5) + floor(n/25) + floor(n/125) + ...`. For 100!: 20 + 4 = **24 zeros**.

**Example 6 (HCF word problem).** Largest tile that exactly covers a 6 m 30 cm x 4 m 20 cm floor: HCF(630, 420) = **210 cm**.

**Other facts:** the sum of the first n natural numbers is `n(n + 1)/2`; of squares `n(n + 1)(2n + 1)/6`; of cubes `[n(n + 1)/2]²`; a number leaving remainder r when divided by a, b, c is `LCM(a, b, c) k + r`.

---

## A3. Percentages

**Core:** `x% = x/100`. Percentage change = (new - old)/old x 100. **Successive changes** of a% and b%: net = `a + b + ab/100` (use negative for decreases). If A is x% more than B, then B is `x/(100 + x) x 100` % less than A.

**Example 1.** A price rises 20% then falls 20%: net = 20 - 20 + (20)(-20)/100 = **-4%** (a 4% loss; the base changed).

**Example 2.** A's salary is 25% more than B's. B's salary is less than A's by `25/125 x 100 =` **20%**.

**Example 3 (population).** A town of 40,000 grows 10% in year one and 20% in year two: 40,000 x 1.1 x 1.2 = **52,800**.

**Example 4 (marks).** A student needs 40% to pass, scores 120 and fails by 30: pass mark = 150 = 40% → total = **375**.

**Example 5 (expenses).** A man spends 20% on rent and 30% of the remainder on food; savings are what percent? Remaining after rent = 80%; food = 24%; left = **56%**.

**Tip:** work with a base of 100 or 1,000 to avoid fractions.

---

## A4. Profit, loss and discount

**Formulas:** profit % = (SP - CP)/CP x 100 (always on **cost price**); `SP = CP x (100 + p)/100`; discount % = (MP - SP)/MP x 100 (on **marked price**); **successive discounts** a% and b%: `a + b - ab/100`. If CP of x articles = SP of y articles, profit % = `(x - y)/y x 100`.

**Example 1.** CP = 800, marked 25% above cost, 10% discount allowed. MP = 1,000; SP = 900; profit = 100/800 = **12.5%**.

**Example 2 (successive discounts).** 20% then 10%: 20 + 10 - 2 = **28%** equivalent single discount.

**Example 3 (the classic loss trap).** Two articles sold at 600 each, one at 20% profit and one at 20% loss. CP1 = 600/1.2 = 500; CP2 = 600/0.8 = 750; total CP = 1,250; total SP = 1,200; loss = 50, i.e. **4%** (general result: loss % = (x/10)² for equal selling prices and equal x% profit and loss).

**Example 4.** A dealer marks goods 40% above cost and gives 10% discount; profit % = 1.4 x 0.9 = 1.26 → **26%**.

**Example 5.** If the cost of 12 items equals the selling price of 10 items: profit = (12 - 10)/10 = **20%**.

---

## A5. Simple and compound interest

**Formulas:** `SI = PRT/100`; amount with CI: `A = P(1 + r/100)^n`; CI = A - P. For compounding m times a year: `A = P(1 + r/(100m))^(mn)`. For two years, **CI - SI = P(r/100)²**; for three years, CI - SI = `P(r/100)² (3 + r/100)`.

**Example 1.** P = 10,000, r = 10%, 2 years: SI = 2,000; amount (CI) = 10,000 x 1.21 = 12,100, so CI = **2,100**; difference = **100** = 10,000 x (0.1)².

**Example 2 (half-yearly).** 8,000 at 10% p.a. for 1 year compounded half-yearly: rate 5% for 2 periods = 8,000 x 1.05² = **8,820**.

**Example 3 (doubling).** At simple interest a sum doubles in 8 years; rate = 100/8 = **12.5%**; it triples in 16 years (gains 2P in 16 years).

**Example 4.** The CI-SI difference for 2 years at 10% is 50: P = 50/(0.1)² = **5,000**.

**Rule of 72 (estimation):** time to double at r% compound ≈ 72/r years.

---

## A6. Ratio, proportion, mixtures and ages

**Ratio:** `a:b = c:d` if `ad = bc`. Combine ratios by matching the common term (A:B = 2:3 and B:C = 4:5 → multiply to make B equal: A:B:C = 8:12:15). **Partnership:** profit shares are in the ratio of (capital x time).

**Alligation (mixtures):** `cheaper quantity : dearer quantity = (dearer price - mean price) : (mean price - cheaper price)`.

**Example 1.** Mix wine at 40/L and 60/L to get 52/L: ratio = (60 - 52) : (52 - 40) = 8 : 12 = **2 : 3**.

**Example 2 (replacement).** A 40 L milk solution has 10 L removed and replaced by water, once: milk left = 40 x (1 - 10/40) = 30 L. Repeating n times: `milk = V (1 - x/V)^n`.

**Example 3 (ages).** A father is three times his son's age; in 12 years he will be twice as old. Let son = s: 3s + 12 = 2(s + 12) → s = **12**, father = 36.

**Example 4 (partnership).** A invests 3,000 for 12 months and B invests 4,500 for 8 months: ratio = 36,000 : 36,000 = **1 : 1**.

**Example 5 (proportion).** If 8 workers build 12 m of wall in a day, 20 workers build 30 m (direct proportion).

---

## A7. Averages

**Formulas:** average = sum / count; **weighted average** = (sum of weight x value)/total weight; if a new item x is added to n items with average A, new average = `(nA + x)/(n + 1)`; average of an AP = (first + last)/2; average speed over equal distances = `2xy/(x + y)`.

**Example 1.** The average of 5 numbers is 20. After adding one number the average is 22. New total = 6 x 22 = 132; old total = 100; the new number is **32**.

**Example 2 (replacing).** The average age of 10 students is 15. A student aged 12 leaves and a new student joins, making the average 15.2; the new student's age = 12 + 10 x 0.2 = **14**.

**Example 3 (average of consecutive numbers).** Average of the first 10 even numbers = (2 + 20)/2 = **11**.

**Example 4 (weighted).** 30 students average 60 and 20 students average 70: overall = (1,800 + 1,400)/50 = **64**.

---

## A8. Time, speed and distance; trains; boats and streams

**Formulas:** `speed = distance / time`; km/h to m/s: x 5/18; m/s to km/h: x 18/5. **Relative speed:** same direction = difference, opposite = sum. A train of length L crossing a pole takes L/speed; crossing a platform or another object of length P takes (L + P)/speed. **Boats:** downstream = u + v, upstream = u - v (u = still water speed, v = current); still water speed = (down + up)/2, current = (down - up)/2.

**Example 1 (average speed).** Going at 60 km/h and returning at 40 km/h over the same distance: average = 2 x 60 x 40/100 = **48 km/h** (not 50).

**Example 2 (train and pole).** A 150 m train at 54 km/h = 15 m/s: pole = 150/15 = **10 s**; a 250 m platform = (150 + 250)/15 = **26.67 s**.

**Example 3 (opposite trains).** Trains of 100 m and 120 m at 40 and 50 km/h towards each other: relative speed = 90 km/h = 25 m/s; crossing time = 220/25 = **8.8 s**.

**Example 4 (boats).** Boat speed in still water 10 km/h, current 2 km/h: downstream 12, upstream 8. A 24 km trip each way takes 24/12 + 24/8 = 2 + 3 = **5 hours**.

**Example 5 (chase).** A is 10 km behind B; A runs at 6 km/h and B at 4 km/h in the same direction: relative speed 2 km/h, so A catches B in **5 hours**.

**Example 6 (meeting).** Two cities 300 km apart; cars start together at 40 and 60 km/h towards each other: they meet after 300/100 = **3 hours**, 120 km from the first city.

---

## A9. Time and work; pipes and cisterns

**Methods:** (1) **Rate method:** A finishes in `a` days → rate 1/a per day; together rate = sum of rates. (2) **LCM (unit) method:** take total work = LCM of the days; compute units per day. A and B together: `ab/(a + b)` days. Work = men x days x hours x efficiency (`M1 D1 H1 / W1 = M2 D2 H2 / W2`). Pipes: inlet adds, outlet subtracts.

**Example 1.** A does a job in 10 days and B in 15: together `10 x 15/25 =` **6 days**.

**Example 2 (LCM method).** A: 12 days, B: 15 days, C: 20 days. Total work = LCM = 60 units; rates 5, 4, 3 per day = 12 per day → **5 days**.

**Example 3 (leaving midway).** A and B can finish in 6 and 12 days; both work for 2 days, then A leaves. Work done in 2 days = 2 x (1/6 + 1/12) = 1/2; remaining 1/2 by B alone = 6 days; total = **8 days**.

**Example 4 (pipes).** A fills a tank in 6 hours and B empties it in 12 hours; both open: net rate = 1/6 - 1/12 = 1/12 → **12 hours**.

**Example 5 (wages).** Wages are shared in proportion to work done: A works 3 days and B 2 days at equal efficiency; a payment of 500 splits **300 : 200**.

**Example 6 (men-days).** 6 men finish a job in 12 days: man-days = 72; 9 men need 72/9 = **8 days**.

---

## A10. Permutations and combinations

**Formulas:** `nPr = n!/(n - r)!` (order matters); `nCr = n!/(r!(n - r)!)` (order does not matter); `nCr = nC(n - r)`. Arrangements with repeated items: `n!/(p! q! ...)`. **Circular arrangements:** `(n - 1)!` (and `(n - 1)!/2` if clockwise and anticlockwise are the same). **Stars and bars:** distributing n identical items into r boxes (empty allowed): `C(n + r - 1, r - 1)`; each box at least one: `C(n - 1, r - 1)`. Fundamental principle: multiply for steps (AND), add for alternatives (OR).

**Example 1.** Arrangements of the letters of LEVEL: `5!/(2! x 2!) = 120/4 =` **30**.

**Example 2.** Choose a committee of 3 from 6 people: C(6,3) = **20**.

**Example 3 (at least one).** A committee of 3 from 4 men and 3 women with at least one woman: total C(7,3) = 35 minus all men C(4,3) = 4 → **31**.

**Example 4.** Seat 5 people around a round table: (5 - 1)! = **24**.

**Example 5.** 4-digit numbers from the digits 1 to 9 without repetition: 9 x 8 x 7 x 6 = **3,024**.

**Example 6.** Handshakes among 10 people: C(10,2) = **45**.

**Example 7 (stars and bars).** 5 identical balls into 3 boxes: C(7,2) = **21**.

**Example 8 (together).** 5 people where 2 must stand together: treat them as one unit → 4! x 2! = **48**.

---

## A11. Probability

**Formulas:** `P(A) = favourable/total`; `P(not A) = 1 - P(A)`; `P(A or B) = P(A) + P(B) - P(A and B)`; independent events: `P(A and B) = P(A) P(B)`; conditional: `P(A|B) = P(A and B)/P(B)`; **expected value** = sum of value x probability.

**Example 1.** Two dice, sum 7: favourable pairs 6 out of 36 = **1/6**.

**Example 2.** At least one head in 3 tosses = 1 - (1/2)³ = **7/8**.

**Example 3.** A bag has 5 red and 3 blue balls; two are drawn without replacement; both red: C(5,2)/C(8,2) = 10/28 = **5/14**.

**Example 4 (conditional).** A card drawn from a standard deck is a face card; probability it is a king: 4/12 = **1/3**.

**Example 5.** Expected value of one throw of a fair die: (1 + 2 + 3 + 4 + 5 + 6)/6 = **3.5**.

**Example 6 (with replacement vs without).** Two aces from a deck: without replacement (4/52)(3/51) = **1/221**; with replacement (1/13)² = 1/169.

**Tip:** "at least one" problems are almost always easier as `1 - P(none)`.

---

## A12. Series and sequences

**AP:** `a_n = a + (n - 1)d`, `S_n = n/2 [2a + (n - 1)d]`. **GP:** `a_n = a r^(n-1)`, `S_n = a (r^n - 1)/(r - 1)`, infinite sum `a/(1 - r)` for |r| < 1. Number-series strategies: differences, second differences, ratios, squares/cubes, alternating patterns, prime numbers, Fibonacci-type (sum of previous two).

**Example 1.** Sum of the first 50 natural numbers = 50 x 51/2 = **1,275**.

**Example 2.** AP 3, 7, 11, ...: 20th term = 3 + 19 x 4 = **79**.

**Example 3.** GP 2, 4, 8, ..., 1024 (10 terms): S = 2(2¹⁰ - 1)/(2 - 1) = **2,046**.

**Example 4.** Series 2, 6, 12, 20, 30, ?: differences 4, 6, 8, 10, next 12 → **42** (it is n(n + 1)).

**Example 5.** Series 1, 1, 2, 3, 5, 8, ?: Fibonacci → **13**.

**Example 6.** Sum of the first 20 odd numbers = 20² = **400**.

---

## A13. Algebra and mensuration

**Quadratics:** `ax² + bx + c = 0` has roots `[-b ± √(b² - 4ac)]/2a`; sum of roots = `-b/a`, product = `c/a`; discriminant `b² - 4ac` decides real/equal/complex roots. **Identities:** `(a + b)² = a² + 2ab + b²`, `a³ + b³ = (a + b)(a² - ab + b²)`. **Inequalities:** reverse the sign when multiplying by a negative number. **Linear equations:** substitute or eliminate.

**Example 1.** `x² - 5x + 6 = 0` → roots 2 and 3; sum 5 = -(-5)/1 and product 6 = 6/1 ✓.

**Example 2 (age-style equation).** Two numbers sum to 30 and differ by 6: (30 + 6)/2 = **18** and 12.

**Mensuration formulas**

| Shape | Area / surface | Volume |
| --- | --- | --- |
| Rectangle | l x b |  |
| Triangle | 1/2 x base x height; Heron: √\[s(s - a)(s - b)(s - c)\] |  |
| Circle | πr²; circumference 2πr |  |
| Cube | 6a²; diagonal a√3 | a³ |
| Cuboid | 2(lb + bh + hl); diagonal √(l² + b² + h²) | lbh |
| Cylinder | curved 2πrh; total 2πr(r + h) | πr²h |
| Cone | curved πrl (l = slant height); total πr(r + l) | (1/3)πr²h |
| Sphere | 4πr² | (4/3)πr³ |

**Example 3.** A cylinder with r = 7 and h = 10: V = (22/7) x 49 x 10 = **1,540**.

**Example 4.** A right triangle with legs 3 and 4: hypotenuse 5, area **6**. Pythagorean triples: (3,4,5), (5,12,13), (8,15,17), (7,24,25).

**Example 5.** If a cube's side doubles, surface area becomes **4x** and volume **8x**.

---

## A14. Data interpretation

**Approach:** read titles, units and scale first; decide what is asked (value, ratio, percentage, growth); **estimate** instead of exact long division; use fractions of totals. **Formulas:** percentage = part/whole x 100; **pie chart angle = percentage x 3.6°** (360/100); growth % = (new - old)/old x 100; CAGR = `(end/start)^(1/n) - 1`; average = total/count; ratio comparison by cross-multiplication.

**Example (table).** Sales (in thousands): Jan 40, Feb 50, Mar 45, Apr 60.

- Growth Jan to Feb = 10/40 = **25%**; Feb to Mar = -5/50 = **-10%**.
- Average of four months = 195/4 = **48.75**.
- April's share of the total (195) = 60/195 ≈ **30.8%**, which on a pie chart is about **111°**.
- CAGR over 3 intervals from 40 to 60: (1.5)^(1/3) - 1 ≈ **14.5%**.

**Traps:** mixing percentage points and percent change (a rise from 20% to 25% is 5 percentage points but a 25% increase); wrong base for a percentage; ignoring units (thousands vs millions); reading the wrong axis.

---

## A15. Logical reasoning

**Number and letter series, coding-decoding:** look for constant differences, alternating steps, position values (A=1 ... Z=26), reversed alphabet (A↔Z), shifts.

**Example (coding).** If each letter moves one step forward, APPLE becomes BQQMF; so MANGO becomes **NBOHP**.

**Syllogisms:** draw Venn diagrams. "All A are B; some B are C" gives **no definite conclusion** about A and C (possible, not certain). "All pens are books; all books are bags" → **all pens are bags**.

**Blood relations:** draw a family tree; track gender and generation. *Example:* "Pointing to a man, Ravi says: he is the son of my mother's only brother." Mother's brother = maternal uncle; his son = **cousin**.

**Direction sense:** draw coordinates (N up, E right). *Example:* walk 5 km north, 3 km east, 5 km south: you are **3 km east** of the start. *Example:* 10 m north, turn right (east) 6 m, turn right (south) 10 m: you are **6 m east** of the start.

**Seating arrangement:** list persons, positions and every condition; fix the sure facts first (a definite position), place conditional ones with cases, eliminate contradictions; for circular tables note whether people face the centre (their left/right is reversed relative to the observer).

**Puzzles and data sufficiency:** build a table/grid, mark certain facts, derive; for "is this sufficient" questions test each statement alone and together.

**Clocks and calendars:** the angle between the hands = `|30H - 5.5M|` degrees; the hands overlap about every 65 5/11 minutes; leap year if divisible by 4 (but centuries only if divisible by 400); the odd-days method for the weekday of a date.

---

## A16. Verbal ability (quick wins)

- **Subject-verb agreement:** "Each of the boys **is**" (each, every, neither, one take singular verbs); collective nouns usually singular; with "either...or / neither...nor" the verb agrees with the nearer subject.
- **Tenses:** past perfect for the earlier of two past actions; avoid mixing tenses in one sentence unless the time changes.
- **Prepositions and articles:** fixed pairs ("interested **in**", "good **at**", "different **from**", "depend **on**"); "a" before consonant sounds, "an" before vowel sounds (an hour, a university).
- **Common errors:** pronoun case (between you and **me**), parallel structure ("I like reading and writing"), comparative vs superlative, redundant words.
- **Vocabulary:** learn roots, prefixes and suffixes; prefer words in context (read editorials), use elimination for synonyms/antonyms.
- **Reading comprehension:** read the questions first, skim the passage for structure (first and last lines of paragraphs), answer from the text only (not outside knowledge), and eliminate extreme options ("always", "never").
- **Para-jumbles:** find the opening sentence (introduces the topic, has no pronoun that needs a reference), then connectors and pronouns, then the conclusion.

---

## A17. Practice set (30 questions, timed 35 minutes)

1. 15% of 240 + 25% of 80 = ?
2. A number is increased by 20% and then decreased by 25%. Net change?
3. If the cost price of 12 pens equals the selling price of 10 pens, find the profit percentage.
4. Simple interest on 5,000 at 8% for 3 years?
5. Compound interest on 4,000 at 10% for 2 years?
6. A:B = 2:3 and B:C = 4:5. Find A:C.
7. Average of the first 10 even numbers?
8. A car travels 120 km in 2 hours and then 180 km in 3 hours. Average speed?
9. A can do a job in 12 days and B in 18 days. Together?
10. Pipe A fills a tank in 20 minutes, pipe B in 30 minutes. Both open: time to fill?
11. In how many ways can the letters of MATH be arranged?
12. In how many ways can 5 books be chosen from 8?
13. Probability of a prime number on a fair die?
14. Probability that two cards drawn without replacement from a deck are both aces?
15. Sum of the first 20 odd numbers?
16. Next term: 3, 6, 12, 24, ?
17. Unit digit of 3^45?
18. Trailing zeros in 50!?
19. A 200 m train crosses a pole in 10 s. Speed in km/h?
20. Boat speed in still water 15 km/h, stream 3 km/h. Downstream distance in 2 hours?
21. A 20 L milk-water mixture is in the ratio 4:1. 5 L of water is added. New ratio?
22. The ratio of two ages is 3:5 and their sum is 40. Find the ages.
23. A shopkeeper sells an item at a 10% loss. If he had sold it for 120 more he would have gained 5%. Find the cost price.
24. For what principal is the difference between CI and SI for 2 years at 10% equal to 50?
25. A walks 10 m north, turns right and walks 6 m, then turns right and walks 10 m. Distance from the start?
26. All pens are books; all books are bags. Is the conclusion "all pens are bags" valid?
27. In a pie chart, what angle represents 20%?
28. Sum and product of the roots of 2x² - 10x + 8 = 0?
29. If 40% of a number is 120, what is the number?
30. 6 men finish a job in 12 days. How long will 9 men take?

**Answers and short reasons**

| # | Answer | Reason |
| --- | --- | --- |
| 1 | 56 | 36 + 20 |
| 2 | -10% | 1.2 x 0.75 = 0.9 |
| 3 | 20% | (12 - 10)/10 |
| 4 | 1,200 | 5,000 x 8 x 3/100 |
| 5 | 840 | 4,000 x 1.21 - 4,000 |
| 6 | 8:15 | A:B:C = 8:12:15 |
| 7 | 11 | (2 + 20)/2 |
| 8 | 60 km/h | 300 km / 5 h |
| 9 | 7.2 days | 36/5 (1/12 + 1/18 = 5/36) |
| 10 | 12 min | 1/20 + 1/30 = 1/12 |
| 11 | 24 | 4! |
| 12 | 56 | C(8,5) = C(8,3) |
| 13 | 1/2 | primes 2, 3, 5 out of 6 |
| 14 | 1/221 | (4/52)(3/51) |
| 15 | 400 | 20² |
| 16 | 48 | doubling |
| 17 | 3 | cycle 3, 9, 7, 1; 45 mod 4 = 1 |
| 18 | 12 | 10 + 2 |
| 19 | 72 km/h | 20 m/s x 18/5 |
| 20 | 36 km | (15 + 3) x 2 |
| 21 | 16:9 | milk 16, water 4 + 5 = 9 |
| 22 | 15 and 25 | 40 split 3:5 |
| 23 | 800 | 1.05 CP - 0.9 CP = 120, so CP = 800 |
| 24 | 5,000 | P x 0.01 = 50 |
| 25 | 6 m | 6 m east of the start, same latitude |
| 26 | Yes | transitivity of "all" |
| 27 | 72° | 20 x 3.6 |
| 28 | 5 and 4 | -b/a = 10/2, c/a = 8/2 |
| 29 | 300 | 120/0.4 |
| 30 | 8 days | 6 x 12 = 72 man-days; 72/9 |

**Scoring guide:** 24 or more correct in 35 minutes is a strong placement-test level; 18 to 23 means revise the topics you missed using the tables above; below 18 means rework fundamentals (A1, A3, A4, A8, A9) before timed practice.

**Common mistakes to log:** percentage on the wrong base; mixing average speed with the average of speeds; forgetting to convert units (km/h, m/s, minutes, hours); using nPr when order does not matter; forgetting that probabilities add only for mutually exclusive events; calculating exactly when estimation would eliminate options.

---

*End of Part 7. Part 8 (Linux in depth) follows.*
