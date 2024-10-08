```agda
module CAS706.part1.Decidable where
```

We have a choice as to how to represent relations:
as an inductive data type of _evidence_ that the relation holds,
or as a function that _computes_ whether the relation holds.
Here we explore the relation between these choices.
We first explore the familiar notion of _booleans_,
but later discover that these are best avoided in favour
of a new notion of _decidable_.

## Imports

```agda
import Relation.Binary.PropositionalEquality as Eq
open Eq using (_≡_; refl)
open Eq.≡-Reasoning
open import Data.Nat.Base using (ℕ; zero; suc; _≤_; s≤s; z≤n)
open import Data.Product.Base using (_×_; proj₁; proj₂) renaming (_,_ to ⟨_,_⟩)
open import Data.Sum.Base using (_⊎_; inj₁; inj₂; [_,_]′)
open import Relation.Nullary.Negation as Neg using (¬_)
  renaming (contradiction to ¬¬-intro)
open import Data.Unit using (⊤; tt)
open import Data.Empty using (⊥)
open import Data.Bool.Base using (Bool; true; false)

open import CAS706.part1.Relations using (_<_; z<s; s<s)
```

## Evidence vs Computation

```agda
2≤4 : 2 ≤ 4
2≤4 = s≤s (s≤s z≤n)

¬4≤2 : ¬ (4 ≤ 2)
¬4≤2 (s≤s (s≤s ()))
```

```agda
infix 4 _≤ᵇ_

_≤ᵇ_ : ℕ → ℕ → Bool
zero ≤ᵇ n       =  true
suc m ≤ᵇ zero   =  false
suc m ≤ᵇ suc n  =  m ≤ᵇ n
```

## Relating evidence and computation

map from computation world to the evidence world:
```agda
T : Bool → Set
T true   =  ⊤
T false  =  ⊥

T→≡ : ∀ (b : Bool) → T b → b ≡ true
T→≡ true Tb = refl

≡→T : ∀ {b : Bool} → b ≡ true → T b
≡→T refl = _
```

`T (m ≤ᵇ n)` is inhabited exactly when `m ≤ n` is inhabited.

```agda
≤ᵇ→≤ : ∀ (m n : ℕ) → T (m ≤ᵇ n) → m ≤ n
≤ᵇ→≤ zero n t = z≤n
≤ᵇ→≤ (suc m) (suc n) t = s≤s (≤ᵇ→≤ m n t)
```

```agda
≤→≤ᵇ : ∀ {m n : ℕ} → m ≤ n → T (m ≤ᵇ n)
≤→≤ᵇ z≤n = _
≤→≤ᵇ (s≤s m≤n) = ≤→≤ᵇ m≤n
```

Why different number of clauses?

## The best of both worlds

A function that returns a boolean returns exactly a single bit of information:
does the relation hold or does it not? Conversely, the evidence approach tells
us exactly why the relation holds, but we are responsible for generating the
evidence.  But it is easy to define a type that combines the benefits of
both approaches.  It is called `Dec A`, where `Dec` is short for _decidable_:
```agda
data Dec (A : Set) : Set where
  yes :   A → Dec A
  no  : ¬ A → Dec A
```

Two easy (by now) lemmas that will be useful:
```agda
¬s≤z : ∀ {m : ℕ} → ¬ (suc m ≤ zero)
¬s≤z ()

¬s≤s : ∀ {m n : ℕ} → ¬ (m ≤ n) → ¬ (suc m ≤ suc n)
¬s≤s ¬m≤n (s≤s m≤n) = ¬m≤n m≤n
```
```agda
_≤?_ : ∀ (m n : ℕ) → Dec (m ≤ n)
zero ≤? n = yes z≤n
suc m ≤? zero = no ¬s≤z
suc m ≤? suc n with m ≤? n
... | yes m≤n = yes (s≤s m≤n)
... | no ¬m≤n = no (¬s≤s ¬m≤n)
```

Can use this to _compute_ the evidence: try `2 ≤? 4` and `4 ≤? 2`
(with C-c C-n).
```agda
_ : Dec (2 ≤ 4)
_ = yes (s≤s (s≤s z≤n))

_ : Dec (4 ≤ 2)
_ = no (¬s≤s (¬s≤s ¬s≤z))
```
## Decidables from booleans, and booleans from decidables

Curious readers might wonder if we could reuse the definition of
`m ≤ᵇ n`, together with the proofs that it is equivalent to `m ≤ n`, to show
decidability.
```agda
_≤?′_ : ∀ (m n : ℕ) → Dec (m ≤ n)
m ≤?′ n with m ≤ᵇ n | ≤ᵇ→≤ m n | ≤→≤ᵇ {m} {n}
...        | true   | p        | _            = yes (p tt)
...        | false  | _        | ¬p           = no ¬p
```

The triple binding of the `with` clause in this proof is essential.



Erasure takes a decidable value to a boolean:
```agda
⌊_⌋ : ∀ {A : Set} → Dec A → Bool
⌊ yes x ⌋  =  true
⌊ no ¬x ⌋  =  false
```
Using erasure, we can easily derive `_≤ᵇ_` from `_≤?_`:
```agda
_≤ᵇ′_ : ℕ → ℕ → Bool
m ≤ᵇ′ n  = ⌊ m ≤? n ⌋
```

Further, if `D` is a value of type `Dec A`, then `T ⌊ D ⌋` is
inhabited exactly when `A` is inhabited:
```agda
toWitness : ∀ {A : Set} {D : Dec A} → T ⌊ D ⌋ → A
toWitness {D = yes x} t = x

fromWitness : ∀ {A : Set} {D : Dec A} → A → T ⌊ D ⌋
fromWitness {D = yes a′} a = tt
fromWitness {D = no ¬a} a = ¬a a
```

Using these, we can easily derive that `T (m ≤ᵇ′ n)` is inhabited
exactly when `m ≤ n` is inhabited:
```agda
≤ᵇ′→≤ : ∀ {m n : ℕ} → T (m ≤ᵇ′ n) → m ≤ n
≤ᵇ′→≤  = toWitness

≤→≤ᵇ′ : ∀ {m n : ℕ} → m ≤ n → T (m ≤ᵇ′ n)
≤→≤ᵇ′  = fromWitness
```

In summary, it is usually best to eschew booleans and rely on decidables.

## Logical connectives to Decidable connectives

```agda
infixr 6 _∧_

_∧_ : Bool → Bool → Bool
true  ∧ true  = true
false ∧ _     = false
_     ∧ false = false
```
In Emacs, the left-hand side of the third equation displays in grey,
indicating that the order of the equations determines which of the
second or the third can match.  However, regardless of which matches
the answer is the same.

```agda
infixr 6 _×-dec_

_×-dec_ : ∀ {A B : Set} → Dec A → Dec B → Dec (A × B)
{- first attempt gave a slightly different pattern
yes a ×-dec yes b = yes ⟨ a , b ⟩
yes a ×-dec no ¬b = no λ p → ¬b (proj₂ p)
no ¬a ×-dec dB = no λ { ⟨ a , b ⟩ → ¬a a }

but we can re-order the clauses like so:
i.e. flip last 2 clauses and erase non-needed information
-}
yes a ×-dec yes b = yes ⟨ a , b ⟩
no ¬a ×-dec _ = no λ { ⟨ a , b ⟩ → ¬a a }
_     ×-dec no ¬b = no λ p → ¬b (proj₂ p)
```

```agda
infixr 5 _⊎-dec_

_⊎-dec_ : ∀ {A B : Set} → Dec A → Dec B → Dec (A ⊎ B)
yes a ⊎-dec dB = yes (inj₁ a)
no ¬a ⊎-dec yes b = yes (inj₂ b)
no ¬a ⊎-dec no ¬b = no [ ¬a , ¬b ]′

¬? : ∀ {A : Set} → Dec A → Dec (¬ A)
¬? (yes a) = no λ ¬a → ¬a a
¬? (no ¬a) = yes ¬a
```

```agda
_→-dec_ : ∀ {A B : Set} → Dec A → Dec B → Dec (A → B)
yes a →-dec yes b = yes λ _ → b
yes a →-dec no ¬b = no λ f → ¬b (f a)
no ¬a →-dec dB = yes λ a → Neg.contradiction a ¬a
```

## Proof by reflection {#proof-by-reflection}

Computing p - q
```agda
minus : (p q : ℕ) (q≤p : q ≤ p) → ℕ
minus p zero z≤n = p
minus (suc p) (suc q) (s≤s q≤p) = minus p q q≤p

minus′ : {p q : ℕ} (q≤p : q ≤ p) → ℕ
minus′ {p} z≤n = p
minus′     (s≤s q≤p) = minus′ q≤p
```

Unfortunately, it is painful to use, since we have to explicitly provide
the proof that `n ≤ m`:

```agda
_ : minus 5 3 (s≤s (s≤s (s≤s z≤n))) ≡ 2
_ = refl
```

But we can mine decidability:

```agda
_-_ : (m n : ℕ) {n≤m : T ⌊ n ≤? m ⌋} → ℕ
_-_ m n {n≤m} = minus m n (toWitness n≤m)
```

We can safely use `_-_` as long as we statically know the two numbers:

```agda
_ : 5 - 3 ≡ 2
_ = refl

_ : 12 - 7 ≡ 5
_ = refl
```

If we try to do this, we get yellow, meaning that some information can't be
figured out:

_ : 7 - 12 ≡ {!7 - 12!}
_ = refl

It turns out that this idiom is very common. The standard library defines a
synonym for `T ⌊ ? ⌋` called `True`:

```agda
True : ∀ {Q} → Dec Q → Set
True Q = T ⌊ Q ⌋
```

## Standard Library

```agda
import Data.Bool.Base using (Bool; true; false; T; _∧_; _∨_; not)
import Data.Nat using (_≤?_)
import Relation.Nullary using (Dec; yes; no)
import Relation.Nullary.Decidable using
  (⌊_⌋; True; toWitness; fromWitness; _×-dec_; _⊎-dec_; ¬?)
import Relation.Binary.Definitions using (Decidable)
```

## Unicode

    ∧  U+2227  LOGICAL AND (\and, \wedge)
    ∨  U+2228  LOGICAL OR (\or, \vee)
    ⊃  U+2283  SUPERSET OF (\sup)
    ᵇ  U+1D47  MODIFIER LETTER SMALL B  (\^b)
    ⌊  U+230A  LEFT FLOOR (\clL)
    ⌋  U+230B  RIGHT FLOOR (\clR)
