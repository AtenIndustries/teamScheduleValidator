# Team schedule validator

This validator validates data from a list of schedule entries with the following structure:

- **employeeId**: identifier number for employee
- **team**: name of the team
- **date**: schedule date (ex: 2025-07-19)
- **shift**: working shift (morning, afternoon, night)
- **contractType**: "full-time" or "part-time"
- **pto**: boolean that specifies if a employee is on paid time off during that day

# Project Structure

```
TeamScheduleValidator/
├── TeamScheduleValidator.BAL/             # Business logic layer
│   ├── DTO                                # Data transfer object class definitions
│   ├── Services                           
│   │   └── Interfaces
│   └── Validators
├── TeamScheduleValidator.BAL.Tests/       # Unit tests
└── README.md
```

## Schedule service overview

The schedule service should implement the IValidatableScheduleService interface

```csharp
public interface IValidatableScheduleService
{
    (List<TeamErrorReportDTO> validationErrors, List<ScheduleInputDTO> validEntries) ValidateSchedule(List<ScheduleInputDTO>); 
}
```
`ScheduleInputDTO` describes an entry and follows the same structure as the one presented in the first section of this document
`TeamsErrorReportDTO` describes the error result for a team. Will contain all the entries of that team, and the list of errors (from all levels of erros, as specified in the following section)

# Validation 

This project uses FluentValidation library to build strongly typed validation rules, with a structural and maintainable design. The validations will run on three different levels:
- **Entry**: validates data in a specific entry
- **Employee**: validates data between all entries of an employee
- **Team**: validates data between all entries of a team

## Rules

### Entry level validations
- No shift set when pto is true
- If shift is set, pto should be false

### Employee level validations
- Only one shift per day of work
- No consecutive night shifts
- Part-time employees cannot be in night shift (check if this does not collide with team level validations)

### Team level validations
- Each team must have all shifts filled with at least one employee, except when all employees are on vacation

## Validators configuration

There are 3 levels of validation that bring a challenge if dependency injection is used, 2 of them using a list of data of the same type. In case of using .Net 8+, the best approach is using keyed services and configure them like this in Program.cs

```csharp
builder.Services.AddScoped<IValidator<ScheduleInputDTO>, EntryValidator>();
builder.Services.AddKeyedScoped<IValidator<List<ScheduleInputDTO>>, EmployeeLevelValidator>("EmployeeLevelValidator");
builder.Services.AddKeyedScoped<IValidator<List<ScheduleInputDTO>>, TeamLevelValidator>("TeamLevelValidator");
```

An implementation of `IValidatableScheduleService` would get this validators injected like this

```csharp
public class ValidatableScheduleService
{
    private readonly IValidator<ScheduleInputDTO> _entryValidator;
    private readonly IValidator<List<ScheduleInputDTO>> _employeeValidator;
    private readonly IValidator<List<ScheduleInputDTO>> _teamValidator;

    public MyConsumer(
        IValidator<ScheduleInputDTO>  entryValidator,
        [FromKeyedServices("EmployeeLevelValidator")] IValidator<List<ScheduleInputDTO>> employeeValidator,
        [FromKeyedServices("TeamLevelValidator")] IValidator<List<ScheduleInputDTO>> teamValidator)
    {
        _entryValidator = entryValidator;
        _employeeValidator = employeeValidator;
        _teamValidator = teamValidator;
    }

    public (List<TeamErrorReportDTO> validationErrors, List<ScheduleInputDTO> validEntries) ValidateSchedule(List<ScheduleInputDTO>){....}

}
```


## Algorithm

The algorithm will follow a structure of sorting plus control break. The main loop will take advantage of working with sorted data to aggregate **employee** and **team** schedule entries, and run all validations in all levels in one pass. 

![High level schema](https://i.imageupload.app/068c4440e41e1beb99c8.svg)

In each iteration:

1. Aggregate current entry on the current employee and current team list
2. Run entry level validations an aggregate any entry level error results
3. If all employee entries are aggregated (by check if next entry employee id is different from current):
    1. Run employee validations
    2. Aggregate validations
4. Check if all team entries are aggregated (by check if next entry team name is different from current):
    1. Run team validations
    2. Create team error report with entries and a list of validation errors (entry, employee and team levels)

![Low level schema](https://i.imageupload.app/1b0a4835fb5eca40a8e9.svg)





