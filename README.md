# Team schedule validator

This validator validates data from a list of schedule entries with the following structure:

- **employeeId**: identifier number for employee
- **team**: name of the team
- **date**: schedule date (ex: 2025-07-19)
- **shift**: working shift (morning, afternoon, night)
- **contractType**: "full-time" or "part-time"
- **pto**: boolean that specifies if a employee is on paid time off during that day

# Tech stack
- **.NET Core 10**
- **FluentValidation**
- **xUnit**

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

In this project approach, any validation error in any level will render ALL of the team schedule invalid. This approach avoids inconsistency problems in the final data. 

## Rules

### Entry level validations
- No shift set when pto is true
- If shift is set, pto should be false
- Part-time employees cannot be in night shift (check if this does not collide with team level validations)

### Employee level validations
- Only one shift per day of work
- No consecutive night shifts

### Team level validations
- Each team must have all shifts filled with at least one employee, except when all employees are on vacation

### App settings


## Validators configuration

There are 3 levels of validation that bring a challenge if dependency injection is used, 2 of them using a list of data of the same type. In case of using .Net 8+, the best approach is using keyed services to inject the validators in Program.cs.

```csharp
builder.Services.AddScoped<IValidator<ScheduleInputDTO>, EntryValidator>();
builder.Services.AddKeyedScoped<IValidator<List<ScheduleInputDTO>>, EmployeeLevelValidator>("EmployeeLevelValidator");
builder.Services.AddKeyedScoped<IValidator<List<ScheduleInputDTO>>, TeamLevelValidator>("TeamLevelValidator");
```

An implementation of `IValidatableScheduleService` will then use the keys to properly map the correct validator for each validation level.

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
# Entry level Validator

Implements three specific rules. **shift** must be empty when **pto** is true, but must have a valid value when **pto** is false and **shift** must be valid for the **contractType**.

```csharp
    RuleFor(x => x.Shift).Empty().When(x => x.PTO).WithMessage("Shift must be empty on PTO");
    RuleFor(x => x.Shift).Must(shift => validShifts.Contains(shift)).When(x => !x.PTO).WithMessage("Shift must be set when not on PTO");

    foreach(string contractType in validContractTypes){
        RuleFor(x => x.Shift).Must(shift => contractTypeShifts[contractType].Contains(shift)).When(x=>x.ContractType == contractType && !x.PTO).WithMessage($"Invalid shift {shift} for {contractType} contract");
    }

    RuleFor(x=>x.ContractType).NotEmpty().WithMessage("ContractType must not me empty);
    // Apply not empty rules for the other fields
    ...
```
> [!Note]
> Variables like `validShifts`, `validContractTypes`, `contractTypeShifts` would be loaded from appSettings. In a production environment would be advisable .

# Employee level Validator

Ensures that there is only one shift per day of work and max consecutive days for each contract type.

```csharp
    RuleFor(employeeScheduleList=>employeeScheduleList.Select(employeeScheduleList.Date)).Must(dates=>dates.Count()==dates.Distinct.Count()).WithMessage($"Employee should only have one shift per day of work")
    foreach(string contractType in validContractTypes){
        int maxConsecutiveDays = int.MaxValue;
        maxConsecutiveDaysContractType.TryGetValue(contractType, out maxConsecutiveDays)
        RuleFor(employeeScheduleList=>GetConsecutiveDayCounts(contractType, employeeScheduleList)).Must(counts=>counts.Where(count=>count>maxConsecutiveDays).Count()==0) 
    } 
```
> [!Note]
> `GetConsecutiveDayCounts` will be a function that determines checks if there are consecutive days of the same contract type, and returns the aggregated counts of consecutive days
> The RuleFor for max consecutive days cannot be enclosed in an if based on `TryGetValye` result, because the setting in `maxConsecutiveDaysContractType` can be changed in a production environment with app settings being both set in app and in database


# Team level Validator

Ensures that all shifts are filled if there is a minimum number of employees available. This validation will use dependent rules to avoid conflicts
```csharp
    RuleFor(teamScheduleList=>teamScheduleList.Select(teamScheduleList.Shift).Distinct().Count()).Must(count=>count==validShifts.Length)
        .DependentRules(()=>{
            RuleFor(teamScheduleList=>teamScheduleList).Must(teamScheduleList=>teamScheduleList.Count()!=teamScheduleList.Select(t=>t.PTO).Where(t=>t.PTO).Count())
        });
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





