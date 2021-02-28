# Apuntes-Entity-Framework

- Entity Framework es el ORM(Object-Relational Mapper) de Microsoft, con versiones tanto para la plataforma .NET "tradicional" como para .NET Core (EntityFramework Core).
- Un ORM nos ayuda a mapear las clases de nuestro programa orientado a objetos y las tablas en la base de datos.
- EF usa LINQ
- EF - Migrations
  - Nos ayudan a controlar los diferentes cambios en el modelo que se convertirán en cambios en la base de datos.
  - Al crear una Migración se compara con la última versión del modelo para detectar los cambios

### Paso 1
  - Crear la carpeta models con todos los modelos que quermos convertir a la BBDD

```C#
public class User
{
    public Guid UserId { get; set; }
    public string Name { get: set: }
    public string LastName {get; set;}
    public DateTime DateCreated {get; set;}
    public bool Active {get; set;}
}
```


### Paso 2
  - Crear la carpeta Context y la clse ApiAppContext

```C#
public class ApiAppContext : DbContext
{
    public DbSet<User> Users { get; set; } // Representa la colección de datos que trae de la BBDD, se pone en plural

    public ApiAppContext(DbContextOptions<ApiContext> options) : base(options) { }
    
    protected override void OnModelCreating(ModelBuilder builder) // Método que nos ayuda a crear la base de datos
    {
        base.OnModelCreating(builder);
        
        //Podemos crar una lista de usuarios
        ![image](https://user-images.githubusercontent.com/66184823/109412081-42f7cc00-79a6-11eb-8245-4bf8e897e9b4.png)

        
        builder.Entity<User>().ToTable("User").HasData(Podemos incluir la lista de usuarios); //Creamos una tabla con el modelo User
        builder.Entity<User>().HasKey(p => p.UserId); // Añadimos la clave primaria
    }
}
```

### Paso 3
  - En Startap

```C#
service.AddDbContext<ApiAppContext>(options => options.UseInMemoryDatabase("NombreBBDD")); //Esto crea una BBDD en memoria
```

### Paso 4
  - En el contralador inyectar la dependencia

```C#
private readonly ApiAppContext _apiAppContext;

public UserController(ApiAppContex apiAppContext)
{
    _apiAppContext = apiAppContext;
    _apiAppContext.Database.EnsureCreated(); // Esto es sólo necesario para la BBDD en memoria.
}
```

# Relacionar tablas

- Creamos una tabla de roles

```C#
public class UserRole
{
    public Guid UserRoleId { get; set; }
    public string Role {get;set;}
    public Guid UserId {get;set;}
    public bool Active {get; set;}
    
    public virtual User User {get;set}; //virtual pq es una propiedad que no va a estar llenandose en todo momento y es la que nos permitirá enlazar con la tabla User y traernos los datos de un usuario.
}
```

- En la clase contexto

```C#
public DbSet<UserRole> UserRoles {get;set;}

builder.Entity<UserRole>().ToTable("UserRole").HasKey("UserRoleId");
//Añadimos datos a la tabla UserRole
![image](https://user-images.githubusercontent.com/66184823/109412051-2491d080-79a6-11eb-9763-be6e4e04250f.png)
builder.Entity<UserRole>().HasOne<User>("User"); Relacionamos la tabla UserRole con la tabla User a travñes del UserId
```

- Ahora en el get de UserRole nos traemos los roles y la información de cada usuario relacionado

```C#
[HttpGet]
[Route("GetRoles")]
public ActionResult<IEnumerable<UserRole>> GetRoles()
{
    return _apiAppContext.UserRoles.Include(p => p.User).ToList();
}
```

- En el caso inverso podríamos en usuarios trael la colección de UserRole, para ello en la clase user añadimos

```C#
public virualt ICollection<UserRole> UserRoles {get; set;}
```

- Ahora intetaríamos traernos los datos desde el get de User

```C#
return _apiAppContext.Users.Where(p => p.Active).Include(p => p.UserRoles).ToList();
```

- Pero esto da un error, para solucionaralo tenemos que añadir una configuración a los controladores en Startup.cs

```C#
.AddNewtonsoftJson(options =>
  options.SerializerSettings.ReferenceLoopHandling = ReferenceLoopHandling.Ignore
```

### Si queremos que algún campo venga en el Json (pj active, ya sólo nos vamos a traer lo sque estén activos)

```C#
[JsonIgnore]
public bool Active {get;set;} = true;
```

# Almacenamiento en caché

- Añadimos el servicio

```C#
services.AddResponseCaching(); //Esto tiene configuración para evitar el case sensitive, etc..
```

- Añadimos el middleware

```C#
app.UseResponseCaching();
```

- Añadimos la info al controller en el que queremos almacenar info en caché

```C#
[HttpGet]
[ResponseCache(Duration = 60)] //Podríamos tb poner de que location, etc.. 
```

# Migrations

- Almacena la información de los cambios que hay en el contexto para poder llevar un histórico de los cambios y poder devolverlo a un estado anterior, se require el paquete de EntityFrameworkCore.Desing

```BASH
dotnet ef migrations add InitialCreate
```

- Esto crea una carpeta Migrations con la info, ahora podríamos añadir pj una propiedad nueva Description en la clase UserRole y volver a lanzar la migración.  Ahora podemos actualizar la BBDD con estos cmabios

```BASH
dotnet ef database update
```



