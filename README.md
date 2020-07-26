**Note**: this specification served as a common format for the initial implementation of ranked.vote. Since its creation, the [NIST CVR](https://www.nist.gov/publications/cast-vote-records-common-data-format-specification-version-10) format has come about as a standard format. As such, this format is retired.

# Ballot data format

## Motivation

Ranked-ballot elections produce a lot of useful data, especially when ballot-level data are available. Unfortunately, a roadblock to analysis is that different jurisdictions provide the ballot data in different formats. This document specifies an open format for ranked-ballot elections that is more amenable to analysis and tabulation auditing.

## Design Goals

In making design decisions about the format, the following considerations were prioritized.

- **Compatibility**: As a subset of CSV, the format can be opened directly by any spreadsheet software as well as myriad tools that already deal with CSV data.
- **Jurisdiction-agnosticity**: Any ballot-level ranked-choice election data can be converted to the format.
- **Simplicity**: This document exists to formalize the format, but the format should be simple enough that average users of the data can understand the data by looking at it.
- **Compressibility**: Rather than rely on the data format to bring the size down, the data is repetitive but highly compressible by gzip.
- **Streamability**: Ballots can be processed while streaming the data without loading the whole dataset into memory.

## Specification

*Capitalized advisory words (SHOULD, MUST, etc.) are used in reference to [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).*

Each file represents all ballots voted for a single contest. If an election consists of multiple contests, each contest has its own file.

The format is a subset of comma-separated values as defined in [RFC 4180](https://www.ietf.org/rfc/rfc4180.txt). The text is UTF-8 encoded. The first row of the file is a header row specifying the column names.

The header MUST contain each of the following columns, not necessarily in order.

| column name   | type    |
|---------------|---------|
| `ballot_id`   | string  |
| `rank`        | integer |
| `choice`      | string  |

The file MAY include other columns. These columns are ignored by the standard but allow you to add additional ballot-level metadata.

Each row that follows the header represents one rank-position on one ballot. That is, if a physical ballot ranks five candidates for a particular contest, that ballot corresponds to five different rows in the data format. These rows MUST be consecutive in the output and in increasing order of rank.

## `ballot_id`

Each row associated with the same ballot MUST have the same value in the `ballot_id` column. If an appropriate identifier (e.g. a ballot number) is provided in the raw data, it SHOULD be used verbatim. Otherwise, it is acceptable to generate a unique string for each ballot. It is RECOMMENDED that these strings follow the convention `@1`, `@2`, etc. unless a compelling reason exists to pick a different unique identifier. In any case, it is RECOMMENDED that generated unique identifiers, i.e. those which do not appear in the raw data, begin with the at-sign character (`@`). Readers of the data MUST NOT assume that `ballot_id` values represent integers.

## `rank`

Each row MUST have a positive integer value for `rank`, indicating the rank on the ballot represented by that row. `(ballot_id, rank)` pairs should be unique across all rows. Each ballot MUST have a row for each possible rank. If the physical ballot does not have a vote for a given rank, the `choice` column should be `$UNDERVOTE` in the corresponding row.

## `choice`

The `choice` column MUST contain a non-empty string corresponding to the choice made on the ballot and rank that the row corresponds to. It is RECOMMENDED that the string be the candidate’s name as given in the raw data, but another unique identifier may be used. In the latter case, it is RECOMMENDED that the unique identifier begin with the at-sign (`@`) character, as with the `ballot_id` column. In addition to candidates, the `choice` column may include special values (see below.)

## Special Values of Choices

Several strings are used for special cases:

- `$UNDERVOTE`: the ballot did not list a candidate for this rank.
- `$OVERVOTE`: the ballot voted for multiple candidates in the rank, invalidating the vote for that rank (and possibly others, depending on the jurisdiction’s rules.)
- `$WRITE_IN`: the ballot listed a write-in candidate, and the raw voting data does not indicate that candidate’s name.

## Example Data

The following example shows how the format represents a contest in which one voter ranked the candidates Alice > Bob > Edna and the other voter ranked a write-in candidate first followed by Edna and no vote after that.

    ballot_id,rank,choice
    AB001,1,Alice
    AB001,2,Bob
    AB001,3,Edna
    AB002,1,$WRITE_IN
    AB002,2,Edna
    AB002,3,$UNDERVOTE

## File Extension

The data files should use the standard extension `.csv` to maximize compatibility.

The format is designed to benefit from gzip compression. Consumers and producers MAY support operations directly on gzipped versions of the data with the extension `.csv.gz`.

## Normalized vs. Raw

Raw ballot data may have certain quirks such as the same candidate listed multiple times, overvotes, skipped ranks, etc. that complicate direct analysis of the data. Furthermore, different voting laws may prescribe different treatments of these cases, so certain types of analysis would need to do special preprocessing for different jurisdictions.

To overcome this, we designate a special subset of the ballot data format for **normalized** ranked-choice vote data. This subset adds the additional restrictions:

- Each `(ballot_id, choice)` pair is unique across rows, that is, each ballot lists each candidate at most once.
- For each `ballot_id`, at most one row may have a value of `$UNDERVOTE` or `$OVERVOTE`, and if so that row must be the last (highest-ranked) row.

A general approach to normalization is to, for each ballot, rank the candidates by the order they first appear on that ballot’s ranking, stopping when an `$OVERVOTE` or `$UNDERVOTE` is encountered.

Care should be taken to ensure that the normalization is consistent with voting laws in the relevant jurisdiction. For example, under certain circumstances in Maine, a single undervote may be skipped rather than exhausting the ballot (as the general approach would do.) Normalization must be done in such a way that the tabulation is invariant to the normalization process, i.e. had the physical ballots been cast *as normalized,* the official tabulation would be the same at every round.

In order to distinguish raw and normalized data, the file extensions `.raw.csv` and `.normalized.csv` may be used respectively. The `.normalized.csv` extension SHOULD ONLY be used for data that conforms to the additional restrictions listed in this section.

## Reference Implementation

A reference implementation in Python can be found in the [ranked_vote_tools.format](https://github.com/ranked-vote/ranked-vote-tools/blob/master/ranked_vote/format/__init__.py) package. Importers for several data formats can be found in the [ranked_vote_import](https://github.com/ranked-vote/ranked-vote-import) package.
