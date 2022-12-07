---
layout: post
title:  "Advent of Code Day 1 2022"
date:   2022-12-01 10:02:59 +0100
categories: jekyll update
---
Decided to make this one in cobol.
Very interesting language. Anyone knows how to read variable length numbers?

```cobol
        IDENTIFICATION DIVISION.
        PROGRAM-ID. DAY1.

        ENVIRONMENT DIVISION.
           INPUT-OUTPUT SECTION.
                FILE-CONTROL.
                SELECT CALS ASSIGN TO 'input.txt'
                        ORGANIZATION IS LINE SEQUENTIAL.
        DATA DIVISION.
           FILE SECTION.
           FD CALS.
           01 CALORIES-FILE.
                05 CALORIES PIC 9(5).
           WORKING-STORAGE SECTION.
           01 WS-CALS.
                05 WS-CALORIES PIC 9(5).
           01 WS-CALORIES-SUM PIC 9(08).
           01 WS-CALORIES-MAX PIC 9(08) VALUE 0.
           01 WS-EOF PIC A(1).
        PROCEDURE DIVISION.
           OPEN INPUT CALS.
           PERFORM UNTIL WS-EOF='Y'
                READ CALS INTO WS-CALS
                        AT END MOVE 'Y' TO WS-EOF
                END-READ
                IF WS-CALORIES IS NUMERIC
                        ADD WS-CALORIES TO WS-CALORIES-SUM
                ELSE
                        IF WS-CALORIES-SUM > WS-CALORIES-MAX
                                SET WS-CALORIES-MAX TO WS-CALORIES-SUM
                        END-IF
                        MOVE 0 TO WS-CALORIES-SUM
                END-IF
           END-PERFORM.
           DISPLAY WS-CALORIES-MAX
           CLOSE CALS.
        STOP RUN.
```
