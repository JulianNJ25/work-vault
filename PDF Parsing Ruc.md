
Regex Structure

Fields to get:

- Razon Social

```java
// block of text before Numero Ruc 
private static final Pattern RAZON_SOCIAL_PATTERN =
    Pattern.compile("(?s)(.*?)\\s*Número RUC");
```
- Ruc
```java
private static final Pattern RUC_PATTERN =

Pattern.compile("Número RUC\\s*(\\d{13})"); 
```  

- No. Telefono
```java
private static final Pattern PHONE_PATTERN =
    Pattern.compile("(Celular|Teléfono trabajo):\\s*(\\d+)");
```

- Direccion
```java
private static final Pattern ADDRESS_PATTERN =
    Pattern.compile("Barrio:.*?Referencia:.*?(?=Provincia:)", Pattern.DOTALL);
```

- Nombre Rep Legal
```java
  private static final Pattern REPRESENTANTE_PATTERN =
    Pattern.compile("Representante legal\\s*•\\s*(.+)");
```
  