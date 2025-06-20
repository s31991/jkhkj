

CF




Korzystając z technologii .NET oraz EFCore, utwórz aplikację typu WebApi, której zadaniem
będzie obsługa poniższej bazy danych. Używając podejścia CodeFirst utwórz kontekst
bazy danych, wraz ze wszelkimi potrzebnymi modelami i wykonaj poniższe zadania.
Zadanie 1.
Stwórz endpoint GET:
GET /api/courses
Zwróć listę wszystkich kursów z bazy danych, wraz z informacjami:
• dane zapisanych studentów(imię, nazwisko, email, data zapisu),
• tytuł kursu i nauczycie



[Route("api/[controller]")]
[ApiController]
public class CoursesController : ControllerBase
{
    private readonly AppDbContext _context;
.
    public CoursesController(AppDbContext context)
    {
        _context = context;
    }
.
    [HttpGet]
    public async Task<ActionResult<IEnumerable<object>>> GetCourses()
    {
        var courses = await _context.Courses
            .Include(c => c.Enrollments)
                .ThenInclude(e => e.Student)
            .Select(c => new {
                id = c.Id,
                title = c.Title,
                teacher = c.Teacher,
                students = c.Enrollments.Select(e => new {
                    id = e.Student.Id,
                    firstName = e.Student.FirstName,
                    lastName = e.Student.LastName,
                    email = e.Student.Email,
                    enrollmentDate = e.EnrollmentDate
                }).ToList()
            })
            .ToListAsync();
.
        return Ok(courses);
    }
}




Stwórz endpoint POST:
POST /api/students/with-enrollments
Ten endpoint powinien:
1. Utworzyć nowego studenta na bazie wartości z żądania (zawsze nowego).
2. Dla każdego przekazanego kursu:
a. Jeśli w bazie danych znajduje się już kursu z takią nazwą, creditsami oraz
nauczycielem, nie twórz takiego kursu.
b. Jeśli nie istnieje, dodaj go do bazy.
3. Utworzyć wpisy w tabeli Enrollment, łączące nowego studenta z kursami.


[ApiController]
[Route("api/[controller]")]
public class StudentsController : ControllerBase
{
    private readonly AppDbContext _context;
.
    public StudentsController(AppDbContext context)
    {
        _context = context;
    }
.
    [HttpPost("with-enrollments")]
    public async Task<IActionResult> CreateStudentWithEnrollments([FromBody] CreateStudentWithEnrollmentsDto dto)
    {
        // 1. Utwórz nowego studenta
        var student = new Student
        {
            FirstName = dto.FirstName,
            LastName = dto.LastName,
            Email = dto.Email,
        };
        _context.Students.Add(student);
        await _context.SaveChangesAsync();
.
        // 2. Dla każdego kursu: znajdź lub utwórz
        var enrolledCourses = new List<Course>();
        foreach (var courseDto in dto.Courses)
        {
            var course = await _context.Courses.FirstOrDefaultAsync(c =>
                c.Title == courseDto.Title &&
                c.Credits == courseDto.Credits &&
                c.Teacher == courseDto.Teacher);
.
            if (course == null)
            {
                course = new Course
                {
                    Title = courseDto.Title,
                    Credits = courseDto.Credits,
                    Teacher = courseDto.Teacher
                };
                _context.Courses.Add(course);
                await _context.SaveChangesAsync();
            }
.
            // 3. Dodaj wpis do Enrollment
            var enrollment = new Enrollment
            {
                StudentId = student.Id,
                CourseId = course.Id,
                EnrollmentDate = DateTime.UtcNow
            };
            _context.Enrollments.Add(enrollment);
.
            enrolledCourses.Add(course);
        }
.
        await _context.SaveChangesAsync();
.
        // 4. Odpowiedź
        var response = new
        {
            student = new
            {
                id = student.Id,
                firstName = student.FirstName,
                lastName = student.LastName,
                email = student.Email
            },
            courses = enrolledCourses.Select(c => new
            {
                id = c.Id,
                title = c.Title,
                credits = c.Credits,
                teacher = c.Teacher
            })
        };
.
        return Ok(response);
    }
}


Utwórz endpoint HTTP PUT pod adresem /api/events/{id}, który umożliwi edycję
wydarzenia o podanym id.
Zaktualizować należy:
• Tytuł i opis wydarzenia
• Listę tagów (czyli zapisać nowe powiązania w tabeli EventTags, usuwając stare)
• Listę uczestników (czyli zapisać nowe powiązania w tabeli EventParticipants,
usuwając stare)
Zakładamy, że użytkownik frontendowy wysyła pełny zestaw danych (czyli nowy stan listy
tagów i uczestników), a API aktualizuje odpowiednio obie tabele asocjacyjne


[ApiController]
[Route("api/[controller]")]
public class EventsController : ControllerBase
{
    private readonly AppDbContext _context;
.
    public EventsController(AppDbContext context)
    {
        _context = context;
    }
.
    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateEvent(int id, [FromBody] UpdateEventDto dto)
    {
        var eventEntity = await _context.Events
            .Include(e => e.EventTags)
            .Include(e => e.EventParticipants)
            .FirstOrDefaultAsync(e => e.Id == id);
.
        if (eventEntity == null)
            return NotFound(new { message = "Wydarzenie nie znalezione." });
.
        // Aktualizacja danych podstawowych
        eventEntity.Title = dto.Title;
        eventEntity.Description = dto.Description;
        eventEntity.Date = dto.Date;
.
        // 🧹 Czyścimy stare tagi i uczestników
        _context.EventTags.RemoveRange(eventEntity.EventTags);
        _context.EventParticipants.RemoveRange(eventEntity.EventParticipants);
.
        // ➕ Dodajemy nowe powiązania
        eventEntity.EventTags = dto.TagIds.Select(tagId => new EventTag
        {
            EventId = id,
            TagId = tagId
        }).ToList();
.
        eventEntity.EventParticipants = dto.ParticipantIds.Select(participantId => new EventParticipant
        {
            EventId = id,
            ParticipantId = participantId
        }).ToList();
.
        await _context.SaveChangesAsync();
.
        return Ok(new
        {
            message = "Wydarzenie zaktualizowane pomyślnie.",
            eventId = id,
            updatedTags = dto.TagIds,
            updatedParticipants = dto.ParticipantIds
        });
    }
}







DF 




Utwórz endpoint HTTP GET pod adresem /api/events/details, który zwróci
szczegółowe informacje o wszystkich wydarzeniach.
Zwracane dane powinny zawierać:
• Tytuł i opis wydarzenia
• Datę Wydarzenia
• Nazwę organizatora
• Listę uczestników (ich nazwy użytkownika)
• Listę tagów przypisanych do wydarzenia



using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using EventApi.Models;
using EventApi.DTOs;

namespace EventApi.Controllers;

[ApiController]
[Route("api/[controller]")]
public class EventsController : ControllerBase
{
    private readonly EventDbContext _context;
.
    public EventsController(EventDbContext context)
    {
        _context = context;
    }
.
    [HttpGet("details")]
    public async Task<ActionResult<IEnumerable<EventDetailsDto>>> GetEventDetails()
    {
        var events = await _context.Events
            .Include(e => e.Organizer)
            .Include(e => e.Participants)
            .Include(e => e.Tags)
            .Select(e => new EventDetailsDto
            {
                Id = e.Id,
                Title = e.Title,
                Description = e.Description,
                Date = e.Date,
                Organizer = new UserDto
                {
                    Id = e.Organizer.Id,
                    Username = e.Organizer.Username
                },
                Participants = e.Participants
                    .Select(p => new UserDto
                    {
                        Id = p.Id,
                        Username = p.Username
                    }).ToList(),
                Tags = e.Tags
                    .Select(t => new TagDto
                    {
                        Id = t.Id,
                        Name = t.Name
                    }).ToList()
            })
            .ToListAsync();
.
        return Ok(events);
    }
}



dotnet ef dbcontext scaffold "Server=localhost;Database=EventDb;Trusted_Connection=True;" Microsoft.EntityFrameworkCore.SqlServer -o Models -f


 builder.Services.AddDbContext<EventDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));







dotnet ef migrations add InitialCreate
dotnet ef database update

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer("YourConnectionString"));


public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) {}
.
    public DbSet<Student> Students { get; set; }
    public DbSet<Course> Courses { get; set; }
    public DbSet<Enrollment> Enrollments { get; set; }
.
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Enrollment>()
            .HasKey(e => new { e.StudentId, e.CourseId });
.
        modelBuilder.Entity<Enrollment>()
            .HasOne(e => e.Student)
            .WithMany(s => s.Enrollments)
            .HasForeignKey(e => e.StudentId);
.
        modelBuilder.Entity<Enrollment>()
            .HasOne(e => e.Course)
            .WithMany(c => c.Enrollments)
            .HasForeignKey(e => e.CourseId);
    }
}



