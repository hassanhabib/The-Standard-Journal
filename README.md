# The Standard Journal

# **Contents**
### Api/Controllers

* [Web Api Controllers - no attributes in parameters](#web-api-controllers)


### Blazor
* [Declaration of parameters in Blazor components](#declaration-of-parameters-in-blazor-components)

### Services/Validations
* [Order of Validation functions](#order-of-validation-functions)
* [Validating CreatedBy/UpdatedBy in entities](#createdbyupdatedby-validations)
* [IsDateNotRecent method in validations](#isdatenotrecent-method)
* [Ordering of Validations](#ordering-of-validations)
* [Validations naming conventions - ValidateStudentExists](#validations-naming-conventions)
* [Date Validations - Updated Date/Created Date](#date-validations---updateddatecreateddate)
* [Switch section - case null](#switch-section---null-references)

### Services/Exceptions
* [HttpResponseUrlNotFoundException](#catch-httpresponseurlnotfoundexception)
* [DbUpdateConcurrencyException - Validation Exception](#dbupdateconcurrencyexception---validation-exception)
* [DependencyValidationException](#dependencyvalidationexception)

### Tests/Services/Validations
* [TheoryData in theory xUnit test cases](#theorydata-in-theory-xunit-test-cases)
* [It.IsAny\<T> in exceptions tests](#itisanyt-in-exceptions-tests)

# **6 November 2020**
## Summary
* [Declaration of parameters in Blazor components](#declaration-of-parameters-in-blazor-components)
* [Order of Validation functions](#order-of-validation-functions)

## **Declaration of parameters in Blazor components**
The properties in a Blazor component that accept an parameter/have a parameter attribute are to be declared at the top before any other declarations.

```cs
public partial class TemplateCardsComponent : ComponentBase
{
    [Parameter]
    public List<EmailTemplateView> EmailTemplateViews { get; set; }

    public List<CardBase> EmailTemplateCards { get; set; }
    public List<ButtonBase> EmailTemplateCardButtons { get; set; }

    protected override void OnInitialized()
    {
        CreateEmailTemplateCards();
        CreateEmailTemplateCardButtons();
    }
}
```
## **Order of Validation functions**
Validation functions are first to be ordered according to CRUD, then the level of when they are called by other functions. 

For example, a `WorkItemService` would contain the CRUD functions with validations:
* AddWorkItem
    * ValidateWorkItemOnAdd
* RetrieveAllWorkItems
    * ValidateStorageWorkItems
* RetrieveWorkItemById
    * ValidateWorkItemId
    * ValidateWorkItemExists
* ModifyWorkItem
    * ValidateWorkItemOnModify
    * ValidateStorageWorkItem
* RemoveWorkItem
    * ValidateWorkItemId
    * ValidateWorkItemExists

The ordering within `WorkItemService.Validations` would look like: 

```cs
ValidateWorkItemOnAdd(WorkItem workItem){
    ValidateWorkItem(workItem)
    ValidateWorkItemDates(workItem)
}

ValidateStorageWorkItems
ValidateWorkId
ValidateWorkItemExists
ValidateWorkItemOnModify
ValidateStorageWorkItem

// followed by functions called by the above
ValidateWorkItem
ValidateWorkItemDates

// followed by other functions called by the above
// followed by helper functions
```

# **14 September 2020**
## Summary

* [DbUpdateConcurrencyException - Validation Exception](#dbupdateconcurrencyexception---validation-exception)
* [DependencyValidationException](#dependencyvalidationexception)


## **DbUpdateConcurrencyException - Validation Exception**
DbUpdateConcurrencyException is to be treated as a validation exception instead of dependency exception. 

## **DependencyValidationException**
A ***DependencyValidationException*** wraps validation exceptions from a dependency instead of wrapping them as a dependency exception. Processing and Orchestration services would handle validation exceptions that are propagated from its lower level services as below: 

### **Processing services**

*`EntityProcessingServiceTests.Exceptions.cs`*
```cs
[Fact]
public async Task ShouldThrowDependencyValidationExceptionOnAddIfValidationExceptionOccursAndLogItAsync()
{
    // given
    Entity someEntity = CreateRandomEntity();
    var innerValidationException = new Exception();
    var entityValidationException = new EntityValidationException(innerValidationException);

    var expectedEntityProcessingDependencyValidationException =
        new EntityProcessingDependencyValidationException(innerValidationException);
    ...
}
```

*`EntityProcessingService.Exceptions.cs`*

`TryCatch block`
```cs
catch (EntityValidationException entityValidationException)
{
    throw CreateAndLogDependencyValidationException(entityValidationException);
}
```
```cs
private EntityProcessingDependencyValidationException CreateAndLogDependencyValidationException(
    Exception exception)
{
    var entityProcessingDependencyValidationException =
        new EntityProcessingDependencyValidationException(exception.InnerException);

    this.loggingBroker.LogError(entityProcessingDependencyValidationException);

    return entityProcessingDependencyValidationException;
}
```

### **Orchestration services**
*`EntityOrhcestrationServiceTests.cs`*

```cs
public static TheoryData ProcessingValidationExceptions()
{
    var someException = new Exception();

    return new TheoryData<Exception>
    {
        new EntityProcessingValidationException(someException),
        new EntityProcessingDependencyValidationException(someException)
    };
}
```

*`EntityOrchestrationService.Exceptions.cs`*
```cs
[Theory]
[MemberData(nameof(ProcessingValidationExceptions))]
public async Task ShouldThrowDependencyValidationExceptionOnRemoveIfValidationExceptionOccursAndLogItAsync(
    Exception processingValidationException)
{
    // given
    Guid someEntityId = Guid.NewGuid();

    var expectedEntityOrchestrationDependencyValidationException =
        new EntityOrchestrationDependencyValidationException(
            processingValidationException.InnerException);

    this.entityProcessingServiceMock.Setup(service =>
        service.RemoveEntityByIdAsync(It.IsAny<Guid>()))
            .ThrowsAsync(processingValidationException);
    ...
}

```
*`EntityOrchestrationService.Exceptions.cs`*

`TryCatch block`
```cs
catch (EntityProcessingValidationException entityProcessingValidationException)
{
    throw CreateAndLogDependencyValidationException(entityProcessingValidationException);
}
catch (EntityProcessingDependencyValidationException entityProcessingDependencyValidationException)
{
    throw CreateAndLogDependencyValidationException(entityProcessingDependencyValidationException);
}
```

### **Controller**
*`EntityController.cs`*

`TryCatch block`
```cs
catch (EntityOrchestrationValidationException validationException)
{          
    string innerMessage = ***

    return BadRequest(innerMessage);
}
catch (EntityOrchestrationDependencyValidationException dependencyValidationException)
    when(dependencyValidationException.InnerException is ***Exception)
{          
    string innerMessage = ***

    return Conflict/NotFound/FailedDependency/Locked(innerMessage);
}
catch (EntityOrchestrationDependencyValidationException dependencyValidationException))
{          
    string innerMessage = ***

    return BadRequest(innerMessage);
}
catch (EntityOrchestrationDependencyException dependencyException))
{          
    return Problem(dependencyException.Message);
}
catch (EntityOrchestrationServiceException serviceException))
{          
    return Problem(serviceException.Message);
}
```

# **14 August 2020**
## **Summary**
* [Catch HttpResponseUrlNotFoundException](#catch-httpresponseurlnotfoundexception)
* [TheoryData in theory xUnit test cases](#theorydata-in-theory-xunit-test-cases)
* [It.IsAny\<T> in exceptions tests](#itisanyt-in-exceptions-tests) 

## **Catch HttpResponseUrlNotFoundException**
[RESTFulSense](https://www.nuget.org/packages/RESTFulSense/#) now includes `HttpResponseUrlNotFoundException` which provides a more meaningful name than the generic HttpResponseNotFoundException. Services that call an API using RESTFulSense should account for this exception and log critical. 

*`ServiceTests.Validations.cs`*
```cs
HttpStatusCode notFoundErrorCode = HttpStatusCode.NotFound;
string randomResponseMessage = GetRandomMessage();
string httpResponseExceptionMessage = randomResponseMessage;
var notFoundResponseMessage = new HttpResponseMessage(notFoundErrorCode);

var httpResponseUrlNotFoundException =
    new HttpResponseUrlNotFoundException(
        responseMessage: notFoundResponseMessage,
        message: httpResponseExceptionMessage);
```

*`Service.Exceptions.cs`*
```cs
catch (HttpResponseUrlNotFoundException httpResponseUrlNotFoundException)
{
    throw CreateAndLogCriticalDependencyException(httpResponseUrlNotFoundException);
}
```
## **`TheoryData` in theory xUnit test cases**
Use the strongly typed `TheoryData` instead of `IEnumerable<object[]>`

*`StudentsServiceTests.Validations.cs`*
```cs
public static TheoryData InvalidStringCases()
{
    string noString = null;
    string emptyString = string.Empty;
    string whiteSpaceString = "     ";

    return new TheoryData<string>
    {
        noString,
        emptyString,
        whiteSpaceString
    };
}
```

The `[Theory]` and `[MemberData]` attributes do not change:
```cs
[Theory]
[MemberData(nameof(InvalidStringCases))]
public async Task ShouldThrowValidationExceptionOnAddIfNameIsInvalidAndLogItAsync(string invalidString)
```

Previously: 
```cs
public static IEnumerable<object[]> InvalidStringCases()
{
    string emptyString = string.Empty;
    string noString = null;
    string whiteSpaceString = "     ";

    return new List<object[]>
    {
        new object[] { emptyString },
        new object[] { noString },
        new object[] { whiteSpaceString }
    };
}
```

## **It.IsAny\<T> in exceptions tests**
In the set up and verify statements of the service mock, instead of passing an input, `It.IsAny<T>` should be used as we want to focus on validating an exception was thrown regardless of how the input parameter came into play.

*`StudentServiceTests.Exceptions.cs`*
```cs
this.storageBrokerMock.Setup(broker =>
    broker.InsertStudentAsync(It.IsAny<Student>))
        .ThrowsAsync(sqlException);

this.storageBrokerMock.Verify(broker =>
    broker.InsertStudentAsync(It.IsAny<Student>)),
        Times.Once);
```
# **4 August 2020**

## Summary

* [Web Api Controllers - no attributes in parameters](#web-api-controllers)
* [Validations naming conventions - ValidateStudentExists](#validations-naming-conventions)
* [Date Validations - Updated Date/Created Date](#date-validations---updateddatecreateddate)
* [Switch section - case null](#switch-section---null-references)

## **Web Api Controllers**

A binding source attribute in the parameter is not required in our action methods as the `[ApiController]` attribute applies inference rules.

*`StudentsController.cs`*
```cs
[HttpPost]
public async ValueTask<ActionResult<Student>> PostStudentAsync(Student student) { ... }
```
The above does not need the attribute `[FromBody]` (where the binding source is Request Body) as it is inferred. 

## **Validations naming conventions**
*`StudentService.Validations.cs`*
| Method name                                              | Purpose                                                                                                   |
| :------------------------------------------------------- | :-------------------------------------------------------------------------------------------------------: |
|  ValidateStudentExists(Student student, Guid studentId); | Checks if student is null after retrieving from storage and throws NotFoundStudentException(studentId);   |

## **Date Validations - UpdatedDate/CreatedDate**
To ensure that the updated date is greater than created date of an entity, we can use `IsDatesOutOfOrder` function:
```cs
private bool IsDatesOutOfOrder(DateTimeOffset shouldComeFirstDate, DateTimeOffset shouldComeLastDate) =>
    shouldComeLastDate >= shouldComeFirstDate;
```

We can then call the above function instead of comparing the dates inline in the switch statement:

```cs
case { } when (IsDatesOutOfOrder(student.UpdatedDate, student.UpdatedDate)):
    throw new InvalidStudentException(
        parameterName: nameof(student.UpdatedDate),
        parameterValue: student.UpdatedDate);
```

## **Switch section - null references**
In the switch statement where we check null values, use: 

*`StudentOrchestrationService.Validations.cs`*
```cs
case null:
    throw new NullStudentException();
```

# **30 July 2020**

## Summary

* [Validating CreatedBy/UpdatedBy in entities](#createdbyupdatedby-validations)
* [IsDateNotRecent method in validations](#isdatenotrecent-method)
* [Ordering of Validations](#ordering-of-validations)


## **CreatedBy/UpdatedBy validations**
For entities with CreatedBy and UpdatedBy, we need to validate that CreatedBy and UpdatedBy are the same when creating a new entity. The validations, validations tests and the filler method need to be updated.

*`StudentServiceTests.Validations.cs`*
```cs
[Fact]
public async Task ShouldThrowValidationExceptionOnCreateIfCreatedByAndUpdatedByNotSameAndLogItAsync()
{
    // given
    Student randomStudent = CreateRandomStudent();
    Guid differentThanCreatedId = Guid.NewGuid();
    Student invalidStudent = randomStudent;
    invalidStudent.UpdatedBy = differentThanCreatedId;

    var invalidStudentException = new InvalidStudentException(
        parameterName: nameof(Student.UpdatedBy),
        parameterValue: invalidStudent.UpdatedBy);

    var expectedStudentValidationException =
        new StudentValidationException(invalidStudentException);

    // when
    ValueTask<Student> createStudentTask =
        this.StudentService.CreateStudentAsync(invalidStudent);

    // then
    await Assert.ThrowsAsync<StudentValidationException>(() => createStudentTask.AsTask());

    this.loggingBrokerMock.Verify(broker =>
        broker.LogError(It.Is(SameExceptionAs(expectedStudentValidationException))),
            Times.Once);

    this.storageBrokerMock.Verify(broker =>
        broker.InsertStudentAsync(It.IsAny<Student>()),
            Times.Never);

    this.loggingBrokerMock.VerifyNoOtherCalls();
    this.storageBrokerMock.VerifyNoOtherCalls();
    this.dateTimeBrokerMock.VerifyNoOtherCalls();
}
```

*`StudentsServiceTests.cs`*

``` cs
private Filler<Student> CreateStudentFiller(DateTimeOffset dates)
{
    Guid profileId = Guid.NewGuid();
    var filler = new Filler<Student>();

    filler.Setup()
        .OnType<DateTimeOffset>().Use(dates)
        .OnProperty(student => student.CreatedBy).Use(profileId)
        .OnProperty(student => student.UpdatedBy).Use(profileId);

    return filler;
}
```
## **IsDateNotRecent method**
New standard:

*`StudentServiceTests.Validations.cs`*
``` cs
private bool IsDateNotRecent(DateTimeOffset date)
{
    TimeSpan oneMinute = TimeSpan.FromMinutes(1);
    DateTimeOffset now = this.dateTimeBroker.GetCurrentDateTime();
    TimeSpan difference = now.Subtract(date);

    return difference.Duration() > oneMinute;
}
```

Previously:

``` cs
private bool IsDateNotRecent(DateTimeOffset dateTime)
{
    int oneMinute = 1;
    DateTimeOffset now = this.dateTimeBroker.GetCurrentDateTime();
    TimeSpan difference = now.Subtract(dateTime);

    return Math.Abs(difference.TotalMinutes) > oneMinute;
}
```

## **Ordering of Validations**
Ordering of validations should look like this:
```cs
private void ValidateStudent(Student student) {
    ValidateStudentId(student);
    ValidateStudentStrings(student);
    ValidateStudentDates(student);
}

private void ValidateStudentId(Student student) { ... }
private void ValidateStudentStrings(Student student) { ... }
private void ValidateStudentDates(Student student) { ... }
```

## **Invalid Enum validations**

Let's assume our Student model looks as follows:

```cs
public class Student 
{
    StudentType Type { get; set; }
}

public enum StudentType
{
	FullTime,
	PartTime
}

## **Invalid Enum validations test*
 ```csharp
[Fact]
public async Task ShouldThrowValidationExceptionOnRegisterIfStudentTypeIsInvalidAndLogItAsync()
{
	// given
	Student randomStudent = CreateRandomStudent();
	Student inputStudent = randomStudent;
	StudentType invalidStudentType = GetInvalidStudentType();
	inputStudent.Type = invalidStudentType;

	var invalidStudentException = new InvalidStudentException(
		parameterName: nameof(Student.Type),
		parameterValue: inputStudent.Type);

	var expectedStudentValidationException =
		new StudentValidationException(invalidStudentException);

	// when
	ValueTask<Student> registerStudentTask =
		this.studentService.RegisterStudentAsync(inputStudent);

	// then
	await Assert.ThrowsAsync<StudentValidationException>(() =>
		registerStudentTask.AsTask());

	this.loggingBrokerMock.Verify(broker =>
		broker.LogError(It.Is(
			SameExceptionAs(expectedStudentValidationException))),
				Times.Once);

	this.storageBrokerMock.Verify(broker =>
		broker.InsertStudentAsync(It.IsAny<Student>()),
			Times.Never);

	this.loggingBrokerMock.VerifyNoOtherCalls();
	this.dateTimeBrokerMock.VerifyNoOtherCalls();
	this.storageBrokerMock.VerifyNoOtherCalls();
}

public static StudentType GetInvalidStudentType()
{
	int randomNumber = GetRandomIntegerNumber();

	while(Enum.IsDefined(typeof(StudentType), randomNumber) is true)
	{
		randomNumber = GetRandomIntegerNumber();
	}

	return (StudentType)randomNumber;
}

public static int GetRandomIntegerNumber() =>
	new IntRange(min: int.MinValue, max: int.MaaxValue).GetValue();

```
