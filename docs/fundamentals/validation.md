---
sidebar_label: Validation
---

# Validation

## Where Should Validation Logic be Implemented? On the Application or Domain Layers?

Although there seems to be <a href="https://softwareengineering.stackexchange.com/questions/314861/in-ddd-is-validation-application-logic-or-domain-logic" target="_blank">some debate about this</a>, the simple answer is both.

Shesha applications tend to be very domain-centric and try to minimise the amount of Application level code (including AppServices) that needs to be hand-written. Instead, Shesha applications try to rely as much as possible on auto-generated APIs, which means that there are no physical AppService classes in which validation logic may be inserted. Over and above this, implementing validation logic on the Domain layer also ensures that as much of the business logic as possible is centralised and minimises the possibility of duplication and potential inconsistencies that may arise from this. The bulk of the validation logic should therefore be implemented in the domain layer.

Notwithstanding the above, where custom APIs have been implemented that accept custom DTOs, it is often quite appropriate and necessary to implement additional validation logic at the Application Layer (within the AppService). In particular, this would be necessary if every application or end-point specific business logic needs to be applied that would not be covered by validation enforced within the Domain Layer.

## Validation Approaches
There are multiple schools of thought on the best ways to implement validation logic (check-out articles [here](https://enterprisecraftsmanship.com/posts/validation-and-ddd/) and [here](https://lostechies.com/jimmybogard/2007/10/24/entity-validation-with-visitors-and-extension-methods/) as there is no clear consensus on this topic.

For the purposes of consistency, the following approaches are recommended for Shesha applications:

| **Validation Approach** | **Description** | **When Should it be Used?** | **Limitations** | **More Info** |
|--|--|--|--|--|
| **Configuration** | Validation rules are specified without any changes to code through the Entity Configurator (This is to be implemented in a future version of Shesha) | When simple validation rules supported by the configurator are required. | Limited by the validation scenarios supported by the configurator. |  |
| **Validation attributes** on domain classes | Validation rules are specified in code by adding ASP. NET Core <a href="https://docs.microsoft.com/en-us/aspnet/core/mvc/models/validation?view=aspnetcore-6.0#validation-attributes" target="_blank">Validation Attributes</a> to the entity properties to be validated | When common single property validation rules need to be specified. Standard attributes supported include: Required, EmailAddress, Range, StringLength, Url, RegularExpression. It is also possible to create <a href="https://docs.microsoft.com/en-us/aspnet/core/mvc/models/validation?view=aspnetcore-6.0#custom-attributes" target="_blank">custom validation attributes</a> for easy reusability of validation logic. | Mostly limited to validation logic that depends on a single property value only. |  |
| Implement **`IValidatableObject`** interface | Validation rules are specified in code by implementing the <a href="https://docs.microsoft.com/en-us/aspnet/core/mvc/models/validation?view=aspnetcore-6.0#ivalidatableobject" target="_blank">IValidatableObject.Validate()</a> method on the entity class.| When complex, 'once-off' validation logic is required, or logic that depends on the value of multiple properties simultaneously, and where such logic needs to be part of the entity class. | Validation logic is 'baked into' the entity class and forces any project using the class to enforce the same validation logic, or otherwise create sub-classes. | See sample code below |
| Implement **FluentValidation** classes |Using <a href="https://docs.abp.io/en/abp/latest/FluentValidation" target="_blank">FluentValidation</a> Library | When validation logic needs to be implemented separately from the domain classes. This is usually when either: 1) Validation logic needs to be added to entities whose source code is not available e.g. when validation needs to be implemented as customisations on top of entity classes imported via Nuget, or, 2) Validation logic contained within the FluentValidation class should be optional (functionality in the future will be provided to allow configurators to enable or disable FluentValidation classes) | Adds a bit more complexity to the solution as new classes need to be created. | <a href="https://fluentvalidation.net/" target="_blank">FluentValidation project web-site</a>
**Prevent domain entities from becoming invalid** in the first place | Entity properties that are at risk of becoming invalid are made read-only by making the relevant property setters private. Properties can then only be updated through methods that enforce the validation logic before updating the properties e.g. `UpdateEmail(string newEmail)` could check if Email address is unique before updating the Email property. This is the approach is better aligned with DDD principles and is promoted by frameworks such as [ABP.IO](https://abp.io/) as illustrated in their BookStore sample app. | This approach is generally not recommended because of the limitations specified in the next column.  <br/>The only case where this approach is recommended is where updating of the property value is NOT expected to be done through a form or CRUD operation, but through an explicit user action on the front-end such as clicking a toolbar button. For example, imagine a Content management application, where `Content` objects have a `Status` property with possible values of `Draft`, `Ready`, `Approved`, `Published`, `Retracted`. In such a case you typically would not want to expose the `Status` property as part of an editable form for the user to be able to update arbitrarily as you would want to ensure specific business logic is applied (e.g. send notifications and ensure approval is not by-passed). Instead, you would probably want to add a 'Publish' toolbar button, which when clicked would call a custom end-point, `/content/{id}/publish`, which then calls the `Content.Publish()` entity method which updates the `Content. Status` after checking that current Status is `Approved`. | Much of the Low-Code features of Shesha rely on the use of standard CRUD based operations. Because this approach by definition deviates from the use of CRUD operations, it would add significantly to the configuration effort. Moreover, similarly to the `IValidatableObject` approach, Validation logic is 'baked into' the entity class and forces any project using the class to enforce the same validation logic, or otherwise create sub-classes. | <a href="https://abp.io/" target="_blank">ABP.IO</a> Bookstore sample app


**Notes:**

- The approaches above are not mutually exclusive. Most projects would be expected to utilise a mix of the above approaches depending on the specific requirements.
- Although validation approaches above are described in the context of the Domain class validation Validation attributes, IValidatableObject and FluentValidation validation approaches may all be employed for the validation of DTOs. See [ABP.IO documentation](https://docs.abp.io/en/abp/latest/Validation) for more information.

## Applying Validation on Auto-generated CRUD APIs

With the exception of the last approach in the table above (which depends on the use of custom APIs), any validation rules specified on an entity will automatically be applied just before a new or updated entity is about to be saved to the database.
If the entity violates any validation rules, the Create(Post) or Update(Put) APIs will throw an `AbpValidationException` and return the appropriate validation information.

## Implementing Validations through FluentValidation
In order for the implemented EntityValidator to trigger as mentioned in [https://docs.fluentvalidation.net/en/latest/](https://docs.fluentvalidation.net/en/latest/). Make sure to include the `AbpFluentValidationModule` in the `WebCoreModule` class of the solution as a dependency.

##### Example
``` csharp
 [DependsOn(
        typeof(AbpFluentValidationModule)
	 )]
    public class WebCoreModule : AbpModule
    {
```

* For DynamicAPIs Create and Update will throw a `AbpValidationException` according to the rules implemented in EntityValidator.
* For CustomAPIs two ways of doing it is either through declaring a new instance of the `EntityValidator`/`EntityDtoValidator` class as and use `Validate()` method as below

##### Example
``` csharp
[HttpPost, Route("CustomAPIMethod")]
public async Task<CustomDto> CustomAPIMethod(CustomInput input)
{
  var validator = new EntityDtoValidator();
  var results = await validator.ValidateAsync(input);

 if (!results.isValid)
    //...throw exception
}
```
Or through dependency injection of `IValidator<EntityValidator>`/`IValidator<EntityDtoValidator>` in the `AppService` class as below

##### Example
``` csharp
/// <summary>
/// 
/// </summary>
[AbpAuthorize]
[ApiVersion("1")]
[Route("api/v{version:apiVersion}/His/[controller]")]
public class AppService : SheshaAppServiceBase
{
   private readonly IValidator<EntityDtoValidator> _validator;

   public AppService(IValidator<EntityDtoValidator> validator)
   {
      _validator = validator;
   }

   [HttpPost, Route("CustomAPIMethod")]
   public async Task<CustomDto> CustomAPIMethod(CustomInput input)
   {
     var results = await _validator.ValidateAsync(input);

     if (!results.isValid)
        //...throw exception
    }
}
```


## Common Custom Validation Sample Code

The sample code below illustrates the implementation of common custom validation use cases based on the `IValidatableObject` approach. It illustrates the following use cases:

- Checking if the current entity being updated is new or an existing entity
- Checking the original value of a property before the update
- Making calls to the database

##### Example
``` csharp
    public class Schedule: FullAuditedEntity<Guid>, IValidatableObject    // Implements IValidatableObject interface to enforce custom validation logic
    {
        ...

        public virtual string Name { get; set; }

        public virtual bool Active { get; set; }

        public virtual IList<Appointments> Appointments { get; set; }

        ...

        /// <summary>
        /// Checks if Schedule.Active property has been set to false, and if so, ensures that there are no pending Appointments for this Schedule.
        /// </summary>
        public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
        {
            if (this.IsTransient())
            {
                // Entity is new

                // Apply validation rules here...
            }
            else
            {
                // Entity is an existing entity in the database and is being updated

                // Checking if the 'Active' property has been set to false, whilst there are still Appointments booked against it.
                if (!this.Active)
                {
                    // Checking if Active property was originally true

                    // Checking which properties were updated and if the Active property is amongst them
                    var sessionProvider = StaticContext.IocManager.Resolve<ISessionProvider>();
                    var dirtyProperties = sessionProvider.Session.GetDirtyProperties(this);
                    var activeProperty = dirtyProperties.Find(p => p.Name == nameof(Active));

                    // Getting the original value of the Active property in case it was updated.
                    var activePropertyOriginalValue = activeProperty?.OldValue as bool?;

                    if (activePropertyOriginalValue == true)
                    {
                        // Active property has indeed been updated from true, to false

                        // Now querying the database to check if there are any appointments still pending against this Schedule before allowing to set to Inactive
                        var appointmentsRepo = StaticContext.IocManager.Resolve<IRepository<Appointment, Guid>>();
                        var numPendingAppointments = appointmentsRepo.Count(appointment => appointment.Schedule.Id == this.Id
                                && appointment.Status == RefListAppointmentStatuses.Booked);

                        if (numPendingAppointments > 0)
                            // Returning validation error.
                            yield return new ValidationResult("Cannot set make a schedule inactive if there are any unfulfilled appointments still pending.");
                    }
                }

                yield return null;
            }
        }
    }
```
