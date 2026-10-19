# EventHive — Student Starter

**Zenith School of AI | Individual Open-Source Evaluation**

EventHive is a Python application for managing campus events, registration counts, attendee records and venue capacity.

## Start here
1. Read STUDENT_PROBLEM_STATEMENT.md.
2. Use Python 3.10+ and Git.
3. Run:
```bash
python -m pip install -r requirements-dev.txt
python -m pytest -q
python -m main
```

The starter contains 110 visible tests: 86 pass and 24 fail on the untouched starter by design. Fix the defects; do not delete, skip, weaken or bypass tests. The final evaluation includes 90 additional private checks, for 200 cases total. Private evaluation material is not included here.

## Project map
- models.py — domain model and field definitions.
- service.py — JSON storage, search/filtering, status and domain operations.
- data/records.json — sample records.
- tests/test_contract.py — visible contract tests.
- .github/workflows/tests.yml — automated checks on pushes and pull requests.
- STUDENT_PROBLEM_STATEMENT.md — task, features and rubric.

Start only when the organiser announces kick-off. Check Actions for the visible test report.
