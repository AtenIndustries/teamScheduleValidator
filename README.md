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
    RuleFor(x => x.Shift).Empty().When(x => x.PTO).WithMessage(o=>$"Shift must be empty on PTO for {o.EmployeeID} at {o.Date:yyyy-MM-dd}");
    RuleFor(x => x.Shift).Must(shift => validShifts.Contains(shift)).When(x => !x.PTO).WithMessage(o=>$"Shift must be set when not on PTO for {o.EmployeeID} at {o.Date:yyyy-MM-dd}");
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
RuleFor(employeeScheduleList => employeeScheduleList)
    .Custom((employeeScheduleList, context) =>
    {
        foreach(var g in employeeScheduleList.GroupBy(e=>e.Date)){
            var date = g.Key;
            if g.Count > maxAllowedShiftsPerDay{
                var e = g.First();
                context.AddFailure(new FluentValidation.Results.ValidationFailure(
                    nameof(employeeScheduleList),
                    $"Employee with id {g.EmployeeId} has more than {maxAllowedShiftsPerDay} shifts at {date:yyyy-MM-dd}."
                ));
            }   
        }
    });

RuleFor(employeeScheduleList => employeeScheduleList)
    .Custom((employeeScheduleList, context) =>
    {  
        var firstSchedule = employeeScheduleList.FirstOrDefault();
        int employeeId = firstSchedule?.EmployeeId ?? 0; 
 
        foreach (string shift in validShifts)
        { 
            if (!maxConsecutiveDaysShift.TryGetValue(shift, out int maxConsecutiveDays))
            {
                maxConsecutiveDays = int.MaxValue;
            }  

            bool invalidCount = !IsBellowMaxConsecutiveDaysForShift(employeeScheduleList, shift, maxConsecutiveDays)

            if (invalidCount ) 
            { 
                context.AddFailure(new FluentValidation.Results.ValidationFailure(
                    nameof(employeeScheduleList),
                    $"Employee with id {employeeId} has more than {maxConsecutiveDays} consecutive days of work for '{shift}' shift."
                ));
            }
        }
    });
```
> [!Note]
> `maxConsecutiveDaysShift` is a map loaded from configuration that sets the maximum consecutive days for shift
> `IsBellowMaxConsecutiveDaysForShift` returns false when there are more than the allowed consecutive days for a given shift


# Team level Validator

Ensures that all shifts are filled if there are employees available.

```csharp
    RuleFor(teamScheduleList=>teamScheduleList)
        .Custom((s, ctx)=> {
            var groupedData = list.GroupBy(t => t.Date)
                               .Select(g => new {
                                   Date = g.Key,
                                   NumDistinctShifts = g.Select(t => t.Shift).Distinct().Count(),
                                   EveryoneOnPto = g.All(e => e.PTO)
                               });
            
            foreach (var g in groupedData)
            {
                if (g.NumDistinctShifts != validShifts.Length && !g.EveryoneOnPto)
                {
                    ctx.AddFailure(new FluentValidation.Results.ValidationFailure(
                        nameof(teamScheduleList),
                        $"Error on {g.Date:yyyy-MM-dd}: {g.NumDistinctShifts} filled shifts (expected: {validShifts.Length})"
                    ));
                }
            }
        }) 
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


## Testing

There are going to be two kinds of tests in this project: Unit and Integrated tests. Unit tests will evaluate the validators individually. Integrated tests will evaluate the ValidatableScheduleService when injected with this validators.
For test purpose, the validators would be injected with settings. Also, for the purpose of this exercise, only specific tests will be described, so won't validate date formats for example.

``` json
{
    "validShifts": ["morning", "afternoon", "night"],
    "validContractTypes": ["full-time", "part-time"],
    "contractTypeShifts":{
        "full-time": ["morning", "afternoon", "night"],
        "part-time": ["morning", "afternoon"]
    },
    "maxConsecutiveDaysShift":{
        "night": 1,
    },
    "maxAllowedShiftsPerDay":1
}
```
    
## Unit tests
The unit tests would be written for each validator, by trying to replicate different error and success scenarios.

### Entry level validator tests

#### Valid entry tests
| Use case | Test scenario | Expected Result |
| :--- | :--- | :--- |
| **Employee on full-time contract working on a night shift** | `{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"night", "contractType":"full-time", "pto":false}` | No validation errors |
| **Employee on full-time contract working on a morning shift** |`{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"morning", "contractType":"full-time", "pto":false}`  | No validation errors  |
| **Employee on part-time contract working on a afternoon shift** |`{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"afternoon", "contractType":"part-time", "pto":false}`  | No validation errors  |
| **Employee on part-time contract working on a afternoon shift** |`{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"afternoon", "contractType":"part-time", "pto":false}`  | No validation errors  |
| **Employee on part-time contract and on pto** |`{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"", "contractType":"part-time", "pto":true}`  | No validation errors  |
| **Employee on full-time contract and on pto** |`{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"", "contractType":"full-time", "pto":true}`  | No validation errors  |

#### Invalid entry tests
| Use case | Test scenario | Expected Result |
| :--- | :--- | :--- |
| **Employee with invalid shift value** | `{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"noon", "contractType":"full-time", "pto":false}` | Should throw validation error "Shift must be set when not on PTO for 1 at 2026-09-24"|
| **Employee with valid shift value when on pto** | `{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"afternoon", "contractType":"full-time", "pto":true}` | Should throw validation error "Shift must be empty on PTO for 1 at 2026-09-24"|
| **Employee with invalid shift value when on pto** | `{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"noon", "contractType":"full-time", "pto":true}` | Should throw validation error "Shift must be empty on PTO for 1 at 2026-09-24"|
| **Employee with night shift on part-time** | `{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"night", "contractType":"part-time", "pto":false}` | Should throw validation error "Invalid shift night for part-time contract"|
| **Employee with night shift on part-time and pto** | `{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"night", "contractType":"part-time", "pto":true}` | Should throw validation errors "Shift must be empty on PTO for 1 at 2026-09-24", "Invalid shift night for part-time contract"|

### Employee level validator tests

| Use case | Test scenario | Expected Result |
| :--- | :--- | :--- |
| **Full-time employee with multiple schedule entries** | <pre>[<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"night", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-25", "shift":"morning", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-26", "shift":"afternoon", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-27", "shift":"afternoon", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-28", "shift":"", "contractType":"full-time", "pto":true}<br>]</pre> | No validation errors |
| **Full-time employee with consecutive night shifts** | <pre>[<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"night", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-25", "shift":"night", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-26", "shift":"afternoon", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-27", "shift":"afternoon", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-28", "shift":"", "contractType":"full-time", "pto":true}<br>]</pre> | Should return validation error "Employee with id 1 has more than 1 consecutive days of work for night shift." |
| **Full-time employee with multiple shifts on same day** | <pre>[<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"morning", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"afternoon", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-26", "shift":"afternoon", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-27", "shift":"afternoon", "contractType":"full-time", "pto":false},<br>  {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-28", "shift":"", "contractType":"full-time", "pto":true}<br>]</pre> | Should return validation error "Employee with id {g.EmployeeId} has more than 1 shifts at 2026-09-24." | 


### Team level validator tests

Test input
``` json
[{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"night", "contractType":"full-time", "pto":false}, 
{"employeeId":3, "team":"FantasticTeam", "date":"2026-09-24", "shift":"afternoon", "contractType":"part-time", "pto":false}
{"employeeId":4, "team":"FantasticTeam", "date":"2026-09-24", "shift":"afternoon", "contractType":"full-time", "pto":false} 
{"employeeId":4, "team":"FantasticTeam", "date":"2026-09-28", "shift":"", "contractType":"full-time", "pto":true}
]
```
| Use case | Test scenario | Expected Result |
| :--- | :--- | :--- |
| **Team with one day fully covered and other day with everyone on pto** | <pre>[{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"night", "contractType":"full-time", "pto":false}, {"employeeId":2, "team":"FantasticTeam", "date":"2026-09-24", "shift":"morning", "contractType":"full-time", "pto":false}, {"employeeId":3, "team":"FantasticTeam", "date":"2026-09-24", "shift":"afternoon", "contractType":"part-time", "pto":false}, {"employeeId":4, "team":"FantasticTeam", "date":"2026-09-24", "shift":"afternoon", "contractType":"full-time", "pto":false}, {"employeeId":1, "team":"FantasticTeam", "date":"2026-09-28", "shift":"", "contractType":"full-time", "pto":true}, {"employeeId":2, "team":"FantasticTeam", "date":"2026-09-28", "shift":"", "contractType":"full-time", "pto":true}, {"employeeId":3, "team":"FantasticTeam", "date":"2026-09-28", "shift":"", "contractType":"part-time", "pto":true}, {"employeeId":4, "team":"FantasticTeam", "date":"2026-09-28", "shift":"", "contractType":"full-time", "pto":true}]</pre> | No validation errors |
| **Team has not all shifts for 2026-09-24** | <pre>[{"employeeId":1, "team":"FantasticTeam", "date":"2026-09-24", "shift":"night", "contractType":"full-time", "pto":false}, {"employeeId":3, "team":"FantasticTeam", "date":"2026-09-24", "shift":"afternoon", "contractType":"part-time", "pto":false}, {"employeeId":4, "team":"FantasticTeam", "date":"2026-09-24", "shift":"afternoon", "contractType":"full-time", "pto":false}, {"employeeId":4, "team":"FantasticTeam", "date":"2026-09-28", "shift":"", "contractType":"full-time", "pto":true}]</pre> | Should return validation error "Error on 2026-09-24: 2 filled shifts (expected: 3)" |

