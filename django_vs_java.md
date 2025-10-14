# 📊 Comparaison concrète — Lecture Excel et persistance en base

## **Scénario : Lire un fichier Excel de ventes et sauvegarder en BDD**

Le fichier Excel contient :  
`date`, `produit`, `quantite`, `prix`

---

## 🐍 **Version Django (Python)**

```python
# models.py
from django.db import models

class Vente(models.Model):
    date = models.DateField()
    produit = models.CharField(max_length=100)
    quantite = models.IntegerField()
    prix = models.DecimalField(max_digits=10, decimal_places=2)

# views.py
import pandas as pd
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['POST'])
def upload_excel(request):
    file = request.FILES['file']
    
    # Lire Excel avec pandas
    df = pd.read_excel(file)
    
    # Créer les objets en masse
    ventes = [
        Vente(
            date=row['date'],
            produit=row['produit'],
            quantite=row['quantite'],
            prix=row['prix']
        )
        for _, row in df.iterrows()
    ]
    
    # Insertion en masse (performant)
    Vente.objects.bulk_create(ventes)
    
    return Response({'message': f'{len(ventes)} ventes importées'})
````

**Total : ~30 lignes de code**

---

## ☕ **Version Java Spring Boot**

```java
// Entity
@Entity
@Table(name = "ventes")
public class Vente {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private LocalDate date;
    
    @Column(length = 100, nullable = false)
    private String produit;
    
    @Column(nullable = false)
    private Integer quantite;
    
    @Column(precision = 10, scale = 2, nullable = false)
    private BigDecimal prix;
    
    public Vente() {}

    public Vente(LocalDate date, String produit, Integer quantite, BigDecimal prix) {
        this.date = date;
        this.produit = produit;
        this.quantite = quantite;
        this.prix = prix;
    }
    
    // Getters & Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public LocalDate getDate() { return date; }
    public void setDate(LocalDate date) { this.date = date; }
    public String getProduit() { return produit; }
    public void setProduit(String produit) { this.produit = produit; }
    public Integer getQuantite() { return quantite; }
    public void setQuantite(Integer quantite) { this.quantite = quantite; }
    public BigDecimal getPrix() { return prix; }
    public void setPrix(BigDecimal prix) { this.prix = prix; }
}

// Repository
@Repository
public interface VenteRepository extends JpaRepository<Vente, Long> {}

// Service
@Service
public class VenteService {

    @Autowired
    private VenteRepository venteRepository;

    public int importerExcel(MultipartFile file) throws IOException {
        List<Vente> ventes = new ArrayList<>();

        try (InputStream is = file.getInputStream();
             Workbook workbook = WorkbookFactory.create(is)) {

            Sheet sheet = workbook.getSheetAt(0);

            // Ignorer la première ligne (headers)
            for (int i = 1; i <= sheet.getLastRowNum(); i++) {
                Row row = sheet.getRow(i);
                if (row == null) continue;

                Cell dateCell = row.getCell(0);
                Cell produitCell = row.getCell(1);
                Cell quantiteCell = row.getCell(2);
                Cell prixCell = row.getCell(3);

                LocalDate date = dateCell.getDateCellValue()
                    .toInstant()
                    .atZone(ZoneId.systemDefault())
                    .toLocalDate();

                String produit = produitCell.getStringCellValue();
                Integer quantite = (int) quantiteCell.getNumericCellValue();
                BigDecimal prix = BigDecimal.valueOf(prixCell.getNumericCellValue());

                ventes.add(new Vente(date, produit, quantite, prix));
            }
        }

        venteRepository.saveAll(ventes);
        return ventes.size();
    }
}

// Controller
@RestController
@RequestMapping("/api")
public class VenteController {

    @Autowired
    private VenteService venteService;

    @PostMapping("/upload-excel")
    public ResponseEntity<?> uploadExcel(@RequestParam("file") MultipartFile file) {
        try {
            int count = venteService.importerExcel(file);
            Map<String, Object> response = new HashMap<>();
            response.put("message", count + " ventes importées");
            return ResponseEntity.ok(response);
        } catch (IOException e) {
            return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Erreur lors de l'import: " + e.getMessage());
        }
    }
}
```

**Total : ~120 lignes de code**
(sans compter les imports et la configuration Maven/Gradle)

---

## ⚖️ **Différences clés**

| Aspect              | Django/Python               | Java Spring Boot                     |
| ------------------- | --------------------------- | ------------------------------------ |
| **Lignes de code**  | ~30                         | ~120                                 |
| **Getters/Setters** | ❌ Non nécessaires           | ✅ Obligatoires (ou Lombok)           |
| **Lecture Excel**   | `pd.read_excel()` (1 ligne) | Apache POI (~30 lignes)              |
| **Typage**          | Dynamique, souple           | Statique, strict                     |
| **Configuration**   | Très minimale               | Application.properties + dépendances |
| **Boilerplate**     | Faible                      | Élevé (Entity, DTO, Service...)      |

---

## 💪 **Avantages Java malgré la verbosité**

✅ **Typage fort** — erreurs détectées à la compilation
✅ **Performance** — plus rapide sur gros volumes
✅ **IDE puissants** — génération automatique du code
✅ **Architecture claire** — séparation stricte des couches
✅ **Lombok** — réduit fortement le boilerplate

---

## ✨ **Avec Lombok (Java)**

```java
@Entity
@Data  // Génère getters/setters/toString/equals/hashCode
@NoArgsConstructor
@AllArgsConstructor
public class Vente {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private LocalDate date;
    private String produit;
    private Integer quantite;
    private BigDecimal prix;
}
```

🟢 Réduit à **~10 lignes**, mais toujours plus verbeux que Python pour la logique métier.

---

## 🧩 **Conclusion**

**Django/Python est moins verbeux car :**

* Pas de getters/setters
* Pandas = lecture Excel ultra simple
* Typage dynamique = peu de déclarations
* Syntaxe concise

**Java est plus verbeux MAIS :**

* Plus sûr (typage fort)
* Performant
* Standard en entreprise

