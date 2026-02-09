---
name: jhipster-backend
description: "JHipster Spring Boot backend development patterns. Use when working with: JHipster entity creation, Spring Data JPA repositories, service layer implementation, REST controllers, Liquibase migrations, MapStruct DTOs/mappers, security configuration (JWT, @PreAuthorize), exception handling, SAP integration, or multi-tenant support. Triggers: 'crea entidad jhipster', 'genera servicio', 'agrega endpoint', database schema changes, Spring Boot microservices."
---

# JHipster Backend Development

## Core Patterns

### 1. Entity Creation
- Follow JHipster conventions: `@Entity`, `@Table`, `AbstractAuditingEntity` for audit fields
- Use proper relationships: `@ManyToOne`, `@OneToMany` with `@JoinColumn`
- Include validation annotations: `@NotNull`, `@Size`, `@Pattern`
- Implement `Serializable` and generate proper `equals()` and `hashCode()`

### 2. Repository Layer
- Extend `JpaRepository<Entity, Long>`
- Use Spring Data JPA query methods
- For complex queries, use `@Query` with JPQL
- Use `Specification` pattern for dynamic filters

### 3. Service Layer Architecture
```java
public interface EntityService {
    EntityDTO save(EntityDTO dto);
    Optional<EntityDTO> findOne(Long id);
    Page<EntityDTO> findAll(Pageable pageable);
    void delete(Long id);
}

@Service
@Transactional
public class EntityServiceImpl implements EntityService {
    private final EntityRepository repository;
    private final EntityMapper mapper;

    public EntityServiceImpl(EntityRepository repository, EntityMapper mapper) {
        this.repository = repository;
        this.mapper = mapper;
    }
}
```

### 4. REST Controller Pattern
```java
@RestController
@RequestMapping("/api")
public class EntityResource {

    private final Logger log = LoggerFactory.getLogger(EntityResource.class);
    private final EntityService entityService;

    @PostMapping("/entities")
    public ResponseEntity<EntityDTO> createEntity(@Valid @RequestBody EntityDTO dto)
        throws URISyntaxException {

        log.debug("REST request to save Entity : {}", dto);
        if (dto.getId() != null) {
            throw new BadRequestAlertException("A new entity cannot already have an ID",
                ENTITY_NAME, "idexists");
        }
        EntityDTO result = entityService.save(dto);
        return ResponseEntity.created(new URI("/api/entities/" + result.getId()))
            .headers(HeaderUtil.createEntityCreationAlert(applicationName, true,
                ENTITY_NAME, result.getId().toString()))
            .body(result);
    }

    @GetMapping("/entities")
    public ResponseEntity<List<EntityDTO>> getAllEntities(Pageable pageable) {
        // Implementation with pagination
    }
}
```

### 5. DTO & Mapper Pattern
```java
public class EntityDTO implements Serializable {
    private Long id;
    @NotNull
    private String name;
}

@Mapper(componentModel = "spring", uses = {RelatedEntityMapper.class})
public interface EntityMapper extends BaseEntityMapper<EntityDTO, Entity> {

    @Mapping(source = "relatedEntity.id", target = "relatedEntityId")
    @Mapping(source = "relatedEntity.name", target = "relatedEntityName")
    EntityDTO toDto(Entity entity);

    @Mapping(source = "relatedEntityId", target = "relatedEntity")
    Entity toEntity(EntityDTO dto);
}
```

### 6. Liquibase Migrations
```xml
<changeSet id="[timestamp]-1" author="jhipster">
    <createTable tableName="entity">
        <column name="id" type="bigint" autoIncrement="true">
            <constraints primaryKey="true" nullable="false"/>
        </column>
        <column name="name" type="varchar(255)">
            <constraints nullable="false"/>
        </column>
        <column name="created_by" type="varchar(50)">
            <constraints nullable="false"/>
        </column>
        <column name="created_date" type="timestamp"/>
        <column name="last_modified_by" type="varchar(50)"/>
        <column name="last_modified_date" type="timestamp"/>
    </createTable>
</changeSet>
```

### 7. Security
- Use `@PreAuthorize("hasAuthority('ROLE_USER')")` for method security
- Configure `SecurityConfiguration.java` for endpoint protection
- JWT tokens for stateless authentication

### 8. Exception Handling
```java
throw new BadRequestAlertException("Error message", "entityName", "errorKey");
throw new EntityNotFoundException("Entity not found");
```

## Best Practices

### Database
- Always use Liquibase for schema changes (never manual SQL)
- Include rollback changesets
- Use meaningful changelog IDs: `YYYYMMDDHHMMSS-sequence`

### Performance
- Use `@Transactional(readOnly = true)` for read operations
- Implement pagination for list endpoints
- Use DTOs to avoid lazy loading issues
- Consider `@Cacheable` for frequent queries

### Testing
```java
@SpringBootTest(classes = YourApp.class)
@AutoConfigureMockMvc
@WithMockUser
class EntityResourceIT {

    @Autowired
    private EntityRepository repository;

    @Autowired
    private MockMvc restEntityMockMvc;

    @Test
    @Transactional
    void createEntity() throws Exception {
        // Integration test implementation
    }
}
```

### Code Organization
- Entities: `domain/`
- DTOs: `service/dto/`
- Mappers: `service/mapper/`
- Services: `service/`
- Repositories: `repository/`
- Controllers: `web/rest/`

## Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| LazyInitializationException | Use DTOs with proper mapping, or `@EntityGraph` for fetch joins |
| Circular dependencies in mappers | Use `@Mapper(uses = {})` carefully, break cycles with custom methods |
| Database collation (MySQL to SQL Server) | Specify collation in Liquibase: `collation="SQL_Latin1_General_CP1_CI_AS"` |
| Audit fields not populating | Extend `AbstractAuditingEntity` and configure `@EnableJpaAuditing` |

## Integration Patterns

### SAP Business One
```java
@Service
public class SapIntegrationService {

    @Value("${sap.service-layer.url}")
    private String sapUrl;

    public void syncToSap(EntityDTO dto) {
        // RestTemplate or WebClient to call SAP Service Layer
        // Handle authentication with session cookies
        // Map JHipster entity to SAP format
    }
}
```

### Multi-tenant Support
```java
@Entity
@Table(name = "entity")
@Where(clause = "branch_id = :branchId")
public class Entity extends AbstractAuditingEntity {
    @Column(name = "branch_id")
    private Long branchId;
}
```

## Related Technologies
Spring Boot 3.x, Spring Data JPA, Liquibase, MapStruct, JWT, Hibernate, PostgreSQL/MySQL/SQL Server
