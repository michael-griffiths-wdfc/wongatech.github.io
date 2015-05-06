---
layout: post
title: Unifying generic type parameters using Mono Cecil
author: julienwetterwald
---
Wonga’s backend is composed of autonomous services communicating via messaging using [NServiceBus](http://nservicebus.com/). All in all, our backend services handle close to 2,000 different message types. Our range of products are offered in many countries, and local regulations impose significant variations. In order to manage complexity, our Continuous Integration system is building, testing and packaging different variants of our backend services for every offering. Consequently, the set of handled message types varies across offerings.

In order to populate a [Configuration Management Database](http://en.wikipedia.org/wiki/CMDB), we want to find all message types handled in an offering. The collected data can later be used to validate configurations (e.g. message routes) or configuring various monitoring systems. In NServiceBus, message handlers implement the IHandleMessages&lt;T&gt; interface, where T is the message type. Hence, in order to identify all the handlers, we have to find all the concrete implementations IHandleMessages&lt;T&gt;.

The obvious solution is to write a simple .NET executable loading all the packaged assemblies into the CLR. Thanks to reflection, we can identify all the subtypes of IHandleMessages&lt;T&gt; quite easily:

<pre><span style="color:#39A3BB;">Directory</span>.GetFiles(path, "*.dll")
  .Select(dll =&gt; Assembly.LoadFrom(dll))
  .SelectMany(<span style="color:#39A3BB;">assembly</span> =&gt; assembly.GetTypes())
  .Where(type =&gt; type.IsClass &amp;&amp; !type.IsAbstract)
  .Where(type =&gt; <span style="color:#0000ff;">typeof</span>(<span style="color:#39A3BB;">IHandleMessages</span>&lt;&gt;).IsAssignableFrom(type));</pre>

However, it is usually a bad idea to use reflection to complete static code analysis; mainly because once an assembly is loaded into an AppDomain it cannot be unloaded. Hence, we use the [Cecil library](http://www.mono-project.com/Cecil) from the Mono project to map the assembly into memory, parse the CIL, and walk the internal tables manually. Many tools use Cecil, including [NDepend](http://www.ndepend.com/).

If we rewrite the expression above using Cecil, we get something that looks quite similar:

<pre><span style="color:#39A3BB;">Directory</span>.GetFiles(path, "*.dll")
  .Select(dll =&gt; <span style="color:#39A3BB;">ModuleDefinition</span>.ReadModule(dll))
  .SelectMany(module =&gt; module.GetTypes())
  .Where(type =&gt; type.IsClass &amp;&amp; !type.IsAbstract)
  .Where(type =&gt; IsSubType(type, type.Module.GetType("NServiceBus", "IHandleMessages`1")));</pre>

It is up to us to implement IsSubType, though:

<pre><span style="color:#0000ff;">bool</span> IsSubType(<span style="color:#39A3BB;">TypeDefinition</span> type, <span style="color:#39A3BB;">TypeReference</span> superType)
{
  <span style="color:#0000ff;">return</span>
      IsType(type, superType) ||
      (type.BaseType != <span style="color:#0000ff;">null</span> &amp;&amp; IsSubType(type.BaseType.Resolve(), superType)) ||
      (type.Interfaces
          .Select(interface =&gt; interface.Resolve())
          .Any(interface =&gt; IsSubType(interface, superType)));
}

<span style="color:#0000ff;">bool</span> IsType(<span style="color:#39A3BB;">TypeReference</span> a, <span style="color:#39A3BB;">TypeReference</span> b)
{
  <span style="color:#0000ff;">return</span> a.Namespace == b.Namespace &amp;&amp; a.Name == b.Name;
}</pre>

The base case checks whether the two types are the same. If they are, then by definition, type is a subtype of superType. Otherwise, we’ve two possible recursion paths. First, we check if type is extending a class. If it is, then we check if the base class is a subtype of superType. Otherwise, we check if any of the implemented interfaces are a subtype of superType.

![](/images/2014-04-02-unifying-generic-type-parameters-using-mono-cecil/blog2-1.png)

Now that our code finds the message handlers packaged in an offering, we can find the handled message types. As previously mentioned, the IHandleMessages&lt;T&gt; interface is parameterized by the message type, so at first glance it looks like it shouldn’t be too complicated.
<pre><span style="color:#39A3BB;">Directory</span>.GetFiles(path, "*.dll")
  .Select(dll =&gt; <span style="color:#39A3BB;">ModuleDefinition</span>.ReadModule(dll))
  .SelectMany(module =&gt; module.GetTypes())
  .Where(type =&gt; type.IsClass &amp;&amp; !type.IsAbstract)
  .Where(type =&gt; IsSubType(type, type.Module.GetType("NServiceBus", "IHandleMessages`1")))
  .Select(GetMessageType)

A simple although incorrect implementation of GetMessageType is the following:

<span style="color:#39A3BB;">TypeReference</span> GetMessageType(<span style="color:#39A3BB;">TypeDefinition</span> handlerDefinition)
{
  TypeReference referenceToIHandleMessages = definition.Interfaces
      .Where(interface =&gt; IsType(
          interface, handler.Module.GetType("NServiceBus", "IHandleMessages`1")))
      .Single();
 <span style="color:#0000ff;"> return</span> ((<span style="color:#39A3BB;">GenericInstanceType</span>)referenceToIHandleMessages).GenericArguments.Single();
}</pre>

We look for IHandleMessages&lt;T&gt; in the interfaces implemented by the handler. Then, we extract the only type argument from the reference to IHandleMessages&lt;T&gt;. Note that in order to get the type argument, we cast the reference to IHandleMessages&lt;T&gt; to a GenericInstanceType. In the picture below, IHandleMessages&lt;T&gt; is a TypeDefinition and IHandleMessages&lt;ILoanApproved&gt; is a GenericInstanceType. (In the definition, T is a generic type parameter, while in the reference ILoanApproved is a generic type argument.)

![](/images/2014-04-02-unifying-generic-type-parameters-using-mono-cecil/blog2-2.png)

Although this implementation of GetMessageType works in the vast majority of the cases, it does not work in the general case. As you have probably noticed, we are assuming that the handler implements IHandleMessages&lt;T&gt; directly. In other words, GetMessageType does not work if the handler either inherits from a class implementing IHandleMessages&lt;T&gt; or implements another interface extending IHandleMessages&lt;T&gt;. Let’s have a look at three examples breaking the current implementation:

![](/images/2014-04-02-unifying-generic-type-parameters-using-mono-cecil/blog2-3.png)

We are trying to solve the general problem, so let’s start by rewriting GetMessageType to delegate to a generic GetActualTypeArgument method.
<pre><span style="color:#39A3BB;">TypeReference</span> GetMessageType(<span style="color:#39A3BB;">TypeDefinition</span> handlerDefinition)
{
  <span style="color:#0000ff;">return</span> GetActualTypeArgument(
      handlerDefinition, handler.Module.GetType("NServiceBus", "IHandleMessages`1"), 0);
}</pre>

GetActualTypeArgument takes three parameters: the subtype, the supertype, and the index of the supertype’s type parameter we want to unify. As IHandleMessages&lt;T&gt; has only one type parameter, the index is 0.

<pre><span style="color:#39A3BB;">TypeReference</span> GetActualTypeArgument(
  <span style="color:#39A3BB;">TypeReference</span> subType, <span style="color:#39A3BB;">TypeReference</span> superType, <span style="color:#0000ff;">int</span> typeParameterIndex)
{
  <span style="color:#39A3BB;">IList</span>&lt;<span style="color:#39A3BB;">TypeReference</span>&gt; linearHierarchy =
      GetLinearHierarchy(subType, superType);
  <span style="color:#39A3BB;">IList</span>&lt;<span style="color:#39A3BB;">TypeParameterInstantiation</span>&gt; chain =
      GroupInTypeParameterInstantiations(typeParameterIndex, linearHierarchy);
  <span style="color:#0000ff;">return</span> Unify(chain);
}</pre>

The method works in three steps. First, we extract the linear hierarchy of type references from the supertype to the subtype. Second, for every type reference in the hierarchy, we resolve its definition and extract type parameter instantiations. Finally, we unify the chain of instantiations to resolve for the type parameter we are looking for.

The implementation of GetLinearHierarchy is quite similar to IsSubType. Its goal is to return only the relevant type references from the hierarchy, e.g. only the types from IHandleMessages&lt;T&gt; to the handler.

<pre><span style="color:#39A3BB;">IList</span>&lt;<span style="color:#39A3BB;">TypeReference</span>&gt; GetLinearHierarchy(<span style="color:#39A3BB;">TypeReference</span> subType, <span style="color:#39A3BB;">TypeReference</span> superType)
{
 <span style="color:#39A3BB;"> TypeDefinition</span> subTypeDefinition = subType.Resolve();

  <span style="color:#39A3BB;">Collection</span>&lt;<span style="color:#39A3BB;">TypeReference</span>&gt; interfaces = subTypeDefinition.Interfaces;
  <span style="color:#0000ff;">foreach</span> (<span style="color:#39A3BB;">TypeReference</span> interface <span style="color:#0000ff;">in</span> interfaces)
  {
      <span style="color:#0000ff;">if</span> (IsType(interface, superType))
      {
          <span style="color:#0000ff;">return new</span> <span style="color:#39A3BB;">List</span>&lt;<span style="color:#39A3BB;">TypeReference</span>&gt;(<span style="color:#0000ff;">new</span> [] { interface, subType });
      }

      <span style="color:#39A3BB;">IList</span>&lt;<span style="color:#39A3BB;">TypeReference</span>&gt; linearHierarchy = GetLinearHierarchy(interface, superType);
      <span style="color:#0000ff;">if</span> (linearHierarchy != <span style="color:#0000ff;">null</span>)
      {
          linearHierarchy.Add(subType);
          <span style="color:#0000ff;">return</span> linearHierarchy;
      }
  }

  <span style="color:#39A3BB;">TypeReference</span> baseType = subTypeDefinition.BaseType;
  <span style="color:#0000ff;">if</span> (baseType == <span style="color:#0000ff;">null</span>)
  {
      <span style="color:#0000ff;">return null</span>;
  }
  <span style="color:#0000ff;">else if</span> (IsType(baseType, superType))
  {
      <span style="color:#0000ff;">return new</span> <span style="color:#39A3BB;">List</span>&lt;<span style="color:#39A3BB;">TypeReference</span>&gt;(<span style="color:#0000ff;">new</span>[] { baseType, subType });
  }
  else
  {
      <span style="color:#39A3BB;">IList</span>&lt;<span style="color:#39A3BB;">TypeReference</span>&gt; linearHierarchy = GetLinearHierarchy(baseType, superType);
      <span style="color:#0000ff;">if</span> (linearHierarchy != <span style="color:#0000ff;">null</span>)
      {
          linearHierarchy.Add(subType);
          <span style="color:#0000ff;">return</span> linearHierarchy;
      }
  }
  <span style="color:#0000ff;">return null</span>;
}</pre>

The next step is to resolve the definition of every type reference in the hierarchy and extract type parameter instantiations. The pictures below demonstrate what we are trying to achieve:

![](/images/2014-04-02-unifying-generic-type-parameters-using-mono-cecil/blog2-4.png)

GroupInTypeParameterInstantiations returns a list of TypeParameterInstantiations, a simple class encapsulating a GenericParameter and a TypeReference. Note that as exemplified in the pictures above, the TypeReference might be anything, including a GenericParameter.

<pre><span style="color:#39A3BB;">IList</span>&lt;<span style="color:#39A3BB;">TypeParameterInstantiation</span>&gt; GroupInTypeParameterInstantiations(
  <span style="color:#0000ff;">int</span> index, <span style="color:#39A3BB;">IList</span>&lt;<span style="color:#39A3BB;">TypeReference</span>&gt; linearHierarchy)
{
  <span style="color:#39A3BB;">TypeReference</span> type = linearHierarchy.First();
  <span style="color:#39A3BB;">List</span>&lt;<span style="color:#39A3BB;">TypeReference</span>&gt; tail = linearHierarchy.Skip(1).ToList();

  <span style="color:#39A3BB;">TypeDefinition</span> typeDefinition = type.Resolve();
  <span style="color:#39A3BB;">GenericParameter</span> genericParameter = typeDefinition.GenericParameters[index];
  <span style="color:#39A3BB;">TypeReference</span> genericArgument = ((<span style="color:#39A3BB;">GenericInstanceType</span>)type).GenericArguments[index];
  <span style="color:#0000ff;">var</span> instantiation = <span style="color:#0000ff;">new</span> TypeParameterInstantiation(genericArgument, genericParameter);

  <span style="color:#0000ff;">if</span> (genericArgument <span style="color:#0000ff;">is</span> <span style="color:#39A3BB;">GenericParameter</span>)
  {
      <span style="color:#39A3BB;">TypeDefinition</span> subTypeDefinition = tail.First().Resolve();
     <span style="color:#0000ff;"> for</span> (<span style="color:#0000ff;">var</span> j = 0; j &lt; subTypeDefinition.GenericParameters.Count; j++)
      {
          <span style="color:#0000ff;">if</span> (subTypeDefinition.GenericParameters[j].FullName == genericArgument.FullName)
          {
              <span style="color:#39A3BB;">IList</span>&lt;<span style="color:#39A3BB;">TypeParameterInstantiation</span>&gt; result =
                  GroupInTypeParameterInstantiations(j, tail);
        result.Add(instantiation);
        <span style="color:#0000ff;">return</span> result;
          }
      }
      <span style="color:#0000ff;">throw new</span> <span style="color:#39A3BB;">Exception</span>();
  }
  <span style="color:#0000ff;">return new</span> <span style="color:#39A3BB;">List</span>&lt;<span style="color:#39A3BB;">TypeParameterInstantiation</span>&gt;(<span style="color:#0000ff;">new</span>[] { instantiation });
}</pre>

GroupInTypeParameterInstantiations start from the top of the hierarchy. We extract the generic argument from the type reference and the generic parameter from the type definition using the index, and store them in a TypeParameterInstantiation.

If the generic argument is a generic parameter as in T := U, then we need to recurse in order to find an instantiation for U. Otherwise, we’re done.

The recursion is a little bit tricky. What we need to do is to first find the index of the type parameter in the next type in the hierarchy. As U is a type parameter, we look at the subtype definition, which is FirstHandler&lt;T, U&gt;.&nbsp; We iterate through the type parameters to find the index of U, which is 1. Then, we call GroupInTypeParameterInstantiations on the rest of the hierarchy.

Now that we have all the instantiations, the last step is to unify them in order to find a substitution for the type parameter we are looking for. This is quite simple, as we just have to return the first generic argument from the chain:
<pre><span style="color:#39A3BB;">TypeReference</span> Unify(<span style="color:#39A3BB;">IList</span>&lt;<span style="color:#39A3BB;">TypeParameterInstantiation</span>&gt; chain)
{
  <span style="color:#0000ff;">return</span> chain.First().GenericArgument;
}</pre>

GetMessageType is now successfully returning the handled message type of handlers that do not implement IHandleMessages&lt;T&gt; directly. There is still one case that is not handled: handlers that inherit from IHandleMessages&lt;T&gt; multiple times. In order to address this issue, one would need to modify GetLinearHierarchy to return multiple hierarchies and then find and unify the instantiations of each linear hierarchy independently.

Finally, it is also worth noting that GetActualTypeArgument doesn’t work if a type argument in the hierarchy is a parameterized type, as in the following example:

![](/images/2014-04-02-unifying-generic-type-parameters-using-mono-cecil/blog2-5.png)

GroupInTypeParameterInstantiations need an extra branch checking if genericArgument is an instance of GenericInstanceType. If it is, then we need to recursively extract all the type parameters from the argument. The complexity arises from the fact that we now have to potentially track multiple type arguments, which makes the implementation of Unify more complicated.

![](/images/2014-04-02-unifying-generic-type-parameters-using-mono-cecil/blog2-6.png)

We will implement these changes in an upcoming blog post. If you want to tackle the challenge on your own, feel free to send your solution to <a href="https://mail.google.com/mail/?view=cm&amp;fs=1&amp;tf=1&amp;to=talent@wonga.com" target="_blank">talent@wonga.com</a>. We would love to talk to you.
