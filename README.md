doctrine-fuzzy
==============

Provides Doctrine DQL fuzzy functions :
- **LEVENSHTEIN**
- **MATCH AGAINST**
- **SOUNDEX**

Installation
============

### Installing with composer

Edit composer.json

```json
    "require": {
    …
        "glanchow/doctrine-fuzzy": "*"
    …
    }
```
Then update

```bash
    php composer.phar update
```

Symfony2 configuration
======================

Edit app/config/config.yml

```yml
doctrine:
    orm:
        entity_managers:
            default:
                dql:
                    numeric_functions:
                        LEVENSHTEIN: WOK\Doctrine\Query\LevenshteinFunction
                        LEVENSHTEIN_RATIO: WOK\Doctrine\Query\LevenshteinRatioFunction
                    string_functions:
                        MATCH: WOK\Doctrine\Query\MatchAgainstFunction
                        SOUNDEX: WOK\Doctrine\Query\SoundexFunction
```

Usage
=====

```php
   /**
    * Suggest Geoname
    */
    public function suggestGeoname($q, $offset, $limit)
    {
        $query = $this->_em
            ->createQueryBuilder('g')
            ->select('g, LEVENSHTEIN(g.name, :q) AS d')
            ->from($this->_entityName, 'g')
            ->orderby('d', 'ASC')
            ->setFirstResult($offset)
            ->setMaxResults($limit)
            ->setParameter('q', $q)
            ->getQuery();

        return $query->getResult();
    }
```

Levenshtein
===========

If Levenshtein functions are not shipped with your database configuration

Start your SQL client

```bash
    mysql
```

Select the database on which you want to add the LEVENSHTEIN functions

```sql
    USE database;
```

Add the functions

```sql
DELIMITER ;;;
CREATE FUNCTION LEVENSHTEIN(s1 VARCHAR(255), s2 VARCHAR(255)) RETURNS int(11) DETERMINISTIC
BEGIN
    DECLARE s1_len, s2_len, i, j, c, c_temp, cost INT;
    DECLARE s1_char CHAR;
    DECLARE cv0, cv1 VARBINARY(256);
    SET s1_len = CHAR_LENGTH(s1), s2_len = CHAR_LENGTH(s2), cv1 = 0x00, j = 1, i = 1, c = 0;
    IF s1 = s2 THEN
        RETURN 0;
    ELSEIF s1_len = 0 THEN
        RETURN s2_len;
    ELSEIF s2_len = 0 THEN
        RETURN s1_len;
    ELSE
        WHILE j <= s2_len DO
            SET cv1 = CONCAT(cv1, UNHEX(HEX(j))), j = j + 1;
        END WHILE;
        WHILE i <= s1_len DO
            SET s1_char = SUBSTRING(s1, i, 1), c = i, cv0 = UNHEX(HEX(i)), j = 1;
            WHILE j <= s2_len DO
                SET c = c + 1;
                IF s1_char = SUBSTRING(s2, j, 1) THEN SET cost = 0; ELSE SET cost = 1; END IF;
                SET c_temp = CONV(HEX(SUBSTRING(cv1, j, 1)), 16, 10) + cost;
                IF c > c_temp THEN SET c = c_temp; END IF;
                SET c_temp = CONV(HEX(SUBSTRING(cv1, j+1, 1)), 16, 10) + 1;
                IF c > c_temp THEN SET c = c_temp; END IF;
                SET cv0 = CONCAT(cv0, UNHEX(HEX(c))), j = j + 1;
            END WHILE;
            SET cv1 = cv0, i = i + 1;
        END WHILE;
    END IF;
    RETURN c;
END
;;;
```

```sql
DELIMITER ;;;
CREATE FUNCTION LEVENSHTEIN_RATIO(s1 VARCHAR(255), s2 VARCHAR(255)) RETURNS int(11) DETERMINISTIC
BEGIN
    DECLARE s1_len, s2_len, max_len INT;
    SET s1_len = LENGTH(s1), s2_len = LENGTH(s2);
    IF s1_len > s2_len THEN SET max_len = s1_len; ELSE SET max_len = s2_len; END IF;
    RETURN ROUND((1 - LEVENSHTEIN(s1, s2) / max_len) * 100);
END
;;;
```
