
# Bulls and Cows SQL Implementation
[![LinkedIn][linkedin-shield]][linkedin-url]
[![Outlook][outlook-shield]][outlook-url]

## [About the Game](https://en.wikipedia.org/wiki/Bulls_and_cows).

The numerical version of the game is usually played with four digits but can be played with any number of digits.

- **Gameplay**: Players write a four-digit secret number with all unique digits. They take turns guessing the opponent's number, receiving feedback on the number of bulls and cows.
  - **Example**:
    - Secret number: 4271
    - Opponent's guess: 1234
    - Feedback: 1 bull and 2 cows (bull: "2", cows: "4" and "1")

- **History**: The game has origins in the 1970s with computer versions appearing on mainframes.

## Project Steps
![image](https://github.com/danjour/Bulls-and-Cows/assets/28869251/673e36b3-ce50-4717-ac03-704fcab1e84f)

### Step 1: Create the `guess` Table
Create a table to store guesses with a unique sequence for each day.
```sql
CREATE TABLE guess (
    guess TEXT,
    data DATE,
    daily_sequence INT, -- To track the number of attempts each day.
    UNIQUE (data, daily_sequence)
);
```

### Step 2: Increment `daily_sequence`
To increment the `daily_sequence`, find the maximum value for the current date and increment it.
```sql
CREATE OR REPLACE FUNCTION calculate_daily_sequence() RETURNS TRIGGER AS $$
DECLARE
    max_sequence INT;
BEGIN
    SELECT COALESCE(MAX(daily_sequence), 0) + 1 INTO max_sequence
    FROM guess
    WHERE data = NEW.data;
    NEW.daily_sequence = max_sequence;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### Step 3: Create a Trigger
Create a trigger to call the function before each insert.
```sql
CREATE TRIGGER set_daily_sequence
BEFORE INSERT ON guess
FOR EACH ROW
EXECUTE FUNCTION calculate_daily_sequence();
```

### Step 4: Insert into `guess` Table
Insert a guess and see the daily sequence in action.
```sql
INSERT INTO guess (guess, data)
VALUES ('42600', current_date::date);
```

Example result:
```
guess     data       daily_sequence
42600     2024-05-27     1
```

### Step 5: Create the `target` Table
Create a table to store the target word.
```sql
CREATE TABLE target (
    target TEXT,
    data DATE
);
```

### Step 6: Generate a Daily Target Word
Generate a daily target word using a seed based on the current day.
```sql
WITH seed AS (
    SELECT EXTRACT(EPOCH FROM CURRENT_DATE)::INT AS date_seed
),
wordle_with_id AS (
    SELECT word, ROW_NUMBER() OVER () AS id
    FROM wordle
),
random_value AS (
    SELECT MOD((SELECT date_seed FROM seed), (SELECT COUNT(*) FROM wordle_with_id)) AS random_index
)
INSERT INTO target (target, data)
SELECT w.word AS target, CURRENT_DATE::DATE AS data
FROM wordle_with_id w
WHERE w.id = (SELECT random_index + 1 FROM random_value) -- +1 because the ROW_NUMBER() starts from 1
AND NOT EXISTS ( -- To ensure only one value per day.
    SELECT 1
    FROM target t
    WHERE t.data = CURRENT_DATE::DATE
);
```

### [Step 7:Compare Guess and Target](https://www.postgresql.org/docs/current/plpgsql-control-structures.html).
Create a function to compare each letter and determine bulls and cows.
```sql
CREATE OR REPLACE FUNCTION compare_numbers(target TEXT, guess TEXT)
RETURNS TEXT AS $$
DECLARE
    result TEXT := '';
    i INT;
    target_char CHAR;
    guess_char CHAR;
    target_counts INT[];
    guess_counts INT[];
BEGIN
    -- Initialize count arrays
    target_counts := ARRAY[0, 0, 0, 0, 0, 0, 0, 0, 0, 0];
    guess_counts := ARRAY[0, 0, 0, 0, 0, 0, 0, 0, 0, 0];

    -- Count occurrences of each digit in the target
    FOR i IN 1..length(target) LOOP
        target_char := substr(target, i, 1);
        target_counts[cast(target_char AS INT) + 1] := target_counts[cast(target_char AS INT) + 1] + 1;
    END LOOP;

    -- Count occurrences of each digit in guess
    FOR i IN 1..length(guess) LOOP
        guess_char := substr(guess, i, 1);
        guess_counts[cast(guess_char AS INT) + 1] := guess_counts[cast(guess_char AS INT) + 1] + 1;
    END LOOP;

    -- Compare digit by digit
    FOR i IN 1..length(target) LOOP
        target_char := substr(target, i, 1);
        guess_char := substr(guess, i, 1);

        IF target_char = guess_char THEN
            result := result || '2';  -- Correct digit in correct position
            target_counts[cast(target_char AS INT) + 1] := target_counts[cast(target_char AS INT) + 1] - 1;
            guess_counts[cast(guess_char AS INT) + 1] := guess_counts[cast(guess_char AS INT) + 1] - 1;
        ELSE
            result := result || '0';  -- Initially assume that the digit is not present
        END IF;
    END LOOP;

    -- Check digits present in the wrong position
    FOR i IN 1..length(target) LOOP
        guess_char := substr(guess, i, 1);
        IF substr(result, i, 1) = '0' THEN  
            IF target_counts[cast(guess_char AS INT) + 1] > 0 THEN
                result := overlay(result placing '1' from i for 1);  -- Correct digit in wrong position
                target_counts[cast(guess_char AS INT) + 1] := target_counts[cast(guess_char AS INT) + 1] - 1;
            END IF;
        END IF;
    END LOOP;

    RETURN result;
END;
$$ LANGUAGE plpgsql;
```

### Step 8: Play the Game
Create the target word and make guesses.
```sql
-- Create the target word
WITH seed AS (
    SELECT EXTRACT(EPOCH FROM CURRENT_DATE)::INT AS date_seed
),
wordle_with_id AS (
    SELECT word, ROW_NUMBER() OVER () AS id
    FROM wordle
),
random_value AS (
    SELECT MOD((SELECT date_seed FROM seed), (SELECT COUNT(*) FROM wordle_with_id)) AS random_index
)
INSERT INTO target (target, data)
SELECT w.word AS target, CURRENT_DATE::DATE AS data
FROM wordle_with_id w
WHERE w.id = (SELECT random_index + 1 FROM random_value)
AND NOT EXISTS (
    SELECT 1
    FROM target t
    WHERE t.data = CURRENT_DATE::DATE
);

-- Make a guess
INSERT INTO guess (guess, data)
VALUES ('42600', current_date::date);

-- Check the result
SELECT a.guess, compare_numbers(a.guess, b.target) AS answer
FROM guess a
LEFT JOIN target b ON a.data = b.data
WHERE a.data = current_date::date
ORDER BY daily_sequence ASC;

-- Example output
guess     answer
42600     01001
42000     22002
42700     22012
42500     22002
42600     22202
```

## Conclusion
I don't know.

Links:
https://www.geeksforgeeks.org/how-to-get-first-character-of-a-string-in-sql/

Inspiration:
https://explainextended.com/2022/01/27/a-good-first-word-for-wordle/

[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=for-the-badge&logo=linkedin&colorB=555
[linkedin-url]: https://www.linkedin.com/in/eduardodanjour/
[facebook-shield]:	https://img.shields.io/badge/Facebook-1877F2?style=for-the-badge&logo=facebook&logoColor=555
[facebook-url]: https://www.facebook.com/eduardo.danjour/
[outlook-shield]:https://img.shields.io/badge/Microsoft_Outlook-0078D4?style=for-the-badge&logo=microsoft-outlook&logoColor=555
[outlook-url]: https://www.facebook.com/eduardo.danjour/
