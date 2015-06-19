---
title: The use cases case study
author: konstantinoskonstantinidis
---
Encapsulating complex domain logic into manageable, easy to maintain pieces is something we tried a couple of different approaches on. Just like every shop, we tried the heavy weight domain models all the way to anemic entities with business logic scattered in our application layer(NServiceBus handlers, in our defense).

### What to improve?

Among the goals of our code and responsibility separation journey were the following items:

* Easy in-memory integration tests
* Solution level separation between domain and application
* Slim application code
* Improved code discovery
* Reduce application to domain coupling
* Ability to deploy the domain in different application setups

### Use cases to the rescue

The approach that worked best for us, was using use cases, aggregate services in a way, that were focused around specific - you guessed it - use cases. Consider the following fictional example:

```csharp
public class AcceptLoanAgreementHandler
{
  private ICustomerBalanceCalculator _customerBalanceCalculator;
  private ILoanAgreementRepository _loanAgreementRepository;

  public void Handle(AcceptLoanAgreementMessage message)
  {
    var aggreement = _loanAgreementRepository.Get(message.LoanAgreementId);

    if(_customerBalanceCalculator.Calculate(agreement.CustomerId) > 0)
    {
      agreement.State = LoanAgreementStates.Accepted;

      Bus.Publish<ILoanAgreementAccepted>(x => x.LoanAgreementId = agreement.Id);
    }
    else
    {
      agreement.State = LoanAgreementStates.Declined;

      Bus.Publish<ILoanAgreementDeclined>(x => x.LoanAgreementId = agreement.Id);
    }

    _loanAgreementRepository.Update(agreement);
  }
}
```

The code above demonstrates some of the issues discussed earlier. The first step is to identify the use case(s) and create the corresponding aggregate services.

```csharp
public interface IAcceptLoanAgreementUseCase
{
  void Execute(Guid agreementId);
}
```

The responsibilities of the use cases as we have used them so far is to interact with lean domain models, handle the logic that crosses boundaries via ports and provide responder interfaces that consumers can implement in order to hook into specific events. Let's assume that the customer balance is provided via a microservice through a port.

```csharp
public class AcceptLoanAgreementUseCase
{
  private ICustomerBalanceCalculator _customerBalanceCalculator;
  private ILoanAgreementRepository _loanAgreementRepository;

  public void Execute(Guid agreementId)
  {
    var aggreement = _loanAgreementRepository.Get(message.LoanAgreementId);

    if(_customerBalanceCalculator.Calculate(agreement.CustomerId) > 0)
    {
      agreement.State = LoanAgreementStates.Accepted;

      //TODO: Notify that the loan agreement got accepted
    }
    else
    {
      agreement.State = LoanAgreementStates.Declined;

      //TODO: Notify that the loan agreement got declined
    }
  }
}
```

### Talking back to the application layer

The use cases so far needed two ways to interact with the world. One is to respond back, so the application layer can handle what happens next(eg an event or a saga completion) and the other is to interact with ports e.g. to take a payment.

Responders, are specific to the use case, so we pass them down the execute method since they are integral parts of this unit of work. Ports on the other hand, can be injected on instantiation via the preferred dependency injection method. The responder parameters passed via the use case were driven mainly by the consumers' requirements. In this case, it's a simple Guid but in other cases it can be a full fledged data structure that can evolve without breaking method signatures.

```csharp
public interface ILoanAgreementAcceptanceResponder
{
  void LoanAgreementAccepted(Guid agreementId);
  void LoanAgreementDeclined(Guid agreementId);
}
```

Thus, our execute method looks like this now:

```csharp
public interface IAcceptLoanAgreementUseCase
{
  void Execute(Guid agreementId, ILoanAgreementAcceptanceResponder responder);
}
```

### Putting it all together

Finally, our use case can now call the appropriate responders in order to notify the outside world. Note that we won't be refactoring the state transition to be in the domain model although arguably it should be.

```csharp
public class AcceptLoanAgreementUseCase
{
  private ICustomerBalanceCalculator _customerBalanceCalculator;
  private ILoanAgreementRepository _loanAgreementRepository;

  public void Execute(Guid agreementId, ILoanAgreementAcceptanceResponder responder)
  {
    var aggreement = _loanAgreementRepository.Get(message.LoanAgreementId);

    if(_customerBalanceCalculator.Calculate(agreement.CustomerId) > 0)
    {
      agreement.State = LoanAgreementStates.Accepted;

      responder.Accepted(agreement.Id);
    }
    else
    {
      agreement.State = LoanAgreementStates.Declined;

      responder.Declined(agreement.Id);
    }

    _loanAgreementRepository.Update(agreement);
  }
}
```

And this allows us to put our NServiceBus handler on a diet. Note how we use the Handler as our responder in order to keep things simple. If you don't want to use interface implementations for your responders you can use Functions as parameters in a sort of a visitor pattern manner instead of declaring full fledged responder types.

```csharp
public class AcceptLoanAgreementHandler : ILoanAgreementAcceptanceResponder
{
  private IAcceptLoanAgreementUseCaseFactory _useCaseFactory;

  public void Handle(AcceptLoanAgreementMessage message)
  {
    _useCaseFactory.Create().Execute(message.LoanAgreementId, this);
  }

  public void ILoanAgreementAcceptanceResponder.Accepted(Guid agreementId)
  {
    Bus.Publish<ILoanAgreementAccepted>(x => x.LoanAgreementId = agreement.Id);
  }

  public void ILoanAgreementAcceptanceResponder.Declined(Guid agreementId)
  {
    Bus.Publish<ILoanAgreementDeclined>(x => x.LoanAgreementId = agreement.Id);
  }
}
```

That concludes our refactoring for this example.

### Testing

Unit testing is a given in our development process but we always wished we [could do more with our in-memory tests](https://vimeo.com/68375232). Having the use cases of a particular domain allows us to wire them up in mockup scenarios that imitate the real world. Granted they aren't complete replacements of real integration tests, they still provided a tremendous value in providing confidence for our developers before pushing their code. This helped us in bringing down the number of tests in the upper slides of the testing pyramid and do so with confidence.

### Inspiration / Credits

A lot of our developers are heavily influenced by [Hexagonal Architecture](http://alistair.cockburn.us/Hexagonal+architecture) and [Clean Coders](https://cleancoders.com/).
