# recall

A spaced-repetition tool that turns question/answer pairs in markdown notes into flashcards and schedules their review.

## Language

**Card**:
One question/answer pair taken from a note file, identified by its normalized question.
_Avoid_: Flashcard, note, item

**Note file**:
A markdown file under the notes root that cards are read from. Never modified by recall.
_Avoid_: Source, document

**Deck**:
The group of cards whose note files sit under the same immediate subfolder of the notes root.
_Avoid_: Category, tag, collection

**Grade**:
The learner's rating of how well they recalled a card: Again, Hard, Good, or Easy.
_Avoid_: Score, rating, ease

**Card state**:
Where a card is in its learning life: New, Learning, Review, or Relearning, plus the scheduling data that goes with it.
_Avoid_: Status, progress

**New card**:
A card that has never been graded.
_Avoid_: Unseen card, fresh card

**Due**:
A card is due when its next review day is today or earlier.
_Avoid_: Overdue, pending

**Review**:
Showing one card's question, revealing its answer, and recording a grade.
_Avoid_: Study, quiz, drill
