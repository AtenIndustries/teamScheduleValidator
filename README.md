# Team schedule validator

This validator validates data from a list of schedule entries with the following structure:

- employeeId: identifier number for employee
- team: name of the team
- date: schedule date (ex: 2025-07-19)
- shift: working shift (morning, afternoon, night)
- contractType: "full-time" or "part-time"
- pto: boolean that specifies if a employee is on paid time off during that day

# Project Structure

```
TeamScheduleValidator/
├── TeamScheduleValidator.BAL/             # Business logic layer
│   ├── DTO
│   ├── Services
│   │   ├── 
├── TeamScheduleValidator.BAL.Tests/       # Unit tests
└── README.md
```

 

# Validation 

This project uses FluentValidation library to build strongly typed validation rules, with a structural and maintainable design. 

# Rules

### Entry level validations
- No shift set when pto is true
- If shift is set, pto should be false

### Employee level validations
- Only one shift per day of work
- No consecutive night shifts
- Part-time employees cannot be in night shift (check if this does not collide with team level validations)

### Team level validations
- Each team must have all shifts filled with at least one employee, except when all employees are on vacation

