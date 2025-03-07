---
title: "CommandText Property (ADO)"
description: "CommandText Property (ADO)"
author: rothja
ms.author: jroth
ms.date: "01/19/2017"
ms.prod: sql
ms.technology: ado
ms.topic: reference
f1_keywords:
  - "Command15::CommandText"
helpviewer_keywords:
  - "CommandText property [ADO]"
apitype: "COM"
---
# CommandText Property (ADO)
Indicates the text of a command to be issued against a provider.  
  
## Settings and Return Values  
 Gets or sets a **String** value that contains a provider command, such as an SQL statement, a table name, a relative URL, or a stored procedure call. The default is the empty string ("").  
  
## Remarks  
 Use the **CommandText** property to set or return the text of a command represented by a [Command](./command-object-ado.md) object. Usually this will be an SQL statement, but can also be any other type of command statement recognized by the provider, such as a stored procedure call. An SQL statement must be of the particular dialect or version supported by the provider's query processor.  
  
 If the [Prepared](./prepared-property-ado.md) property of the **Command** object is set to **True** and the **Command** object is bound to an open connection when you set the **CommandText** property, ADO prepares the query (that is, a compiled form of the query that is stored by the provider) when you call the [Execute](./execute-method-ado-command.md) or [Open](./open-method-ado-connection.md) methods.  
  
 Depending on the [CommandType](./commandtype-property-ado.md) property setting, ADO may alter the **CommandText** property. You can read the **CommandText** property at any time to see the actual command text that ADO will use during execution.  
  
 Use the **CommandText** property to set or return a relative URL that specifies a resource, such as a file or directory. The resource is relative to a location specified explicitly by an absolute URL, or implicitly by an open [Connection](./connection-object-ado.md) object.  
  
> [!NOTE]
>  URLs using the http scheme will automatically invoke the [Microsoft OLE DB Provider for Internet Publishing](../../guide/appendixes/microsoft-ole-db-provider-for-internet-publishing.md). For more information, see [Absolute and Relative URLs](../../guide/data/absolute-and-relative-urls.md).  
  
## Applies To  
 [Command Object (ADO)](./command-object-ado.md)  
  
## See Also  
 [ActiveConnection, CommandText, CommandTimeout, CommandType, Size, and Direction Properties Example (VB)](./activeconnection-commandtext-commandtimeout-commandtype-size-example-vb.md)   
 [ActiveConnection, CommandText, CommandTimeout, CommandType, Size, and Direction Properties Example (VC++)](./activeconnection-commandtext-commandtimeout-commandtype-size-example-vc.md)   
 [Requery Method](./requery-method.md)   
 [ActiveConnection, CommandText, CommandTimeout, CommandType, Size, and Direction Properties Example (JScript)](./activeconnection-commandtext-timeout-type-size-example-jscript.md)