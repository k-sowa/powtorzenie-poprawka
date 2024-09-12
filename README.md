
## Ważne informacje

# Czas
Czas najlepiej przechowywać za pomocą LocalDateTime

Możemy go formatować np za pmocą formattera
```java
LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
```
# Parametry
Jeśli chcemy przekazać parametry do @PostMapping w Springu musimy utworzyć osobną klase ponieważ @BodyRequest przyjmuje tylko jeden parametr np.
```java
@PostMapping("/pixel")
    public ResponseEntity<Void> putPixel(@RequestBody PixelRequest pixelRequest){
        String colorHex = pixelRequest.getColor().trim();
        int rgb = Integer.parseInt(colorHex, 16);
        image.setRGB(pixelRequest.getX(), pixelRequest.getY(),rgb);
        String timestamp = LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        //Wywolanie zapytania do bazy danych za pomocą wcześniej utworzonej metody insertEntry
        databaseManager.insertEntry(pixelRequest.getUuid().toString(), pixelRequest.getX(), pixelRequest.getY(), pixelRequest.getColor(), timestamp);
        return ResponseEntity.ok().build();
    }
```


<details>
  <summary>Struktura PixelRequest</summary>
  
  ```java
  public class PixelRequest {
    private UUID uuid;
    private int x;
    private int y;
    private String color;

    public UUID getUuid() {
        return uuid;
    }

    public void setUuid(UUID uuid) {
        this.uuid = uuid;
    }

    public int getX() {
        return x;
    }

    public void setX(int x) {
        this.x = x;
    }

    public int getY() {
        return y;
    }

    public void setY(int y) {
        this.y = y;
    }

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }
}
```
</details>

# Kody błędów lub sukcesu
Kody przekazujemy za pmmocą ResponseEntity
kody które możemy zwrócić wyglądają następująco
<details>
    <summary> Kody </summary>
    
<ul>
    <li>200 OK: Żądanie zostało pomyślnie przetworzone.</li>
    <li>201 Created: Zasób został pomyślnie utworzony.</li>
    <li>204 No Content: Żądanie zostało pomyślnie przetworzone, ale nie ma treści do zwrócenia.</li>
    <li>400 Bad Request: Żądanie jest niepoprawne lub niekompletne.</li>
    <li>401 Unauthorized: Brak autoryzacji do wykonania żądania.</li>
    <li>403 Forbidden: Brak dostępu do zasobu.</li>
    <li>404 Not Found: Żądany zasób nie został znaleziony.</li>
    <li>500 Internal Server Error: Wystąpił błąd na serwerze.</li>
    <li>302 Found: Przekierowanie do innego URL.</li>
</ul>
    
</details>
Przykład

```java
@RestController
public class ExampleController {

    @GetMapping("/validate")
    public ResponseEntity<String> validate(@RequestParam String data) {
        if (data == null || data.isEmpty()) {
            return new ResponseEntity<>("Invalid data provided", HttpStatus.BAD_REQUEST); // Kod statusu 400
        }
        // Logic for valid data
        return new ResponseEntity<>("Data is valid", HttpStatus.OK); // Kod statusu 200
    }
}
```

Jeśli w ResponseEntity damy void np. ResponseEntity<Void> to możemy zwrócic sam kod błędu np:
```java
return ResponseEntity.ok().build();
```

Tworzenie metody która się automatycznie wywołuje po starcie serwera
```java
@Component
public class ImageInitializer implements CommandLineRunner {

    private final ImageController imageController;

    public ImageInitializer(ImageController imageController) {
        this.imageController = imageController;
    }

    @Override
    public void run(String... args) throws Exception {
        imageController.generateImage();
    }
}

```
W tym wypadku wywołujemy metode generateImage() z klasy imageController.
## Bazy danych
<details>

<summary>Tworzenie bazy danych w kodzie oraz używanie tej bazy danych w innych klasach</summary>
  
```java

//WSZYSTKO ZWIĄZANE Z KODEM BAZ DANYCH MUSI UMIEĆ ZWRACAĆ KODY BŁĘDÓW
//Musi być adnotacja @Component jeśli chcemy używać bazy danych wewnątrz innych klas
@Component
public class DatabaseManager {

    private static final String DB_URL = "jdbc:sqlite:database.db";
    //Nazwa bazy danych
    private static final String CREATE_TABLE = "CREATE TABLE IF NOT EXISTS entry (" +
            "token TEXT NOT NULL," +
            " x INTEGER NOT NULL," +
            " y INTEGER NOT NULL," +
            " color TEXT NOT NULL," +
            " timestamp TEXT NOT NULL)";
    //Zapytanie o stworzenie bazy danych

    //Tworzenie zapytań do późniejszego użycia
    private static final String INSERT_ENTRY_SQL = "INSERT INTO entry (token, x, y, color, timestamp) VALUES(?, ?, ?, ?, ?)";
    //Zapytanie do wprowadzenia danych
    private static final String DELETE_ENTRY_SQL = "DELETE FROM entry WHERE token = ?";
    //Usunięcie wpisu z bazy danych
    private static final String SELECT_ENTRY_SQL = "SELECT token, x, y, color FROM entry ORDER BY timestamp";
    //Jakiś select do bazy

    public DatabaseManager() {
        try (Connection connection = DriverManager.getConnection(DB_URL);
        Statement statement = connection.createStatement()) {
            statement.execute(CREATE_TABLE);
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    //Kod odpowiadający za stworzenie bazy danych jako konstruktor
    
    
    
    public void insertEntry(String token, int x, int y, String color, String timestamp) {
        try (Connection connection = DriverManager.getConnection(DB_URL);
        PreparedStatement preparedStatement = connection.prepareStatement(INSERT_ENTRY_SQL)) {
            preparedStatement.setString(1, token);
            preparedStatement.setInt(2,x);
            preparedStatement.setInt(3,y);
            preparedStatement.setString(4,color);
            preparedStatement.setString(5,timestamp);
            preparedStatement.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    //Stworzenie pierwszego używalnego zapytania do bazy danych w tym wypadku jest to insert
    
    
    public void deleteEntry(String token) {
        try (Connection connection = DriverManager.getConnection(DB_URL);
        PreparedStatement preparedStatement = connection.prepareStatement(DELETE_ENTRY_SQL)) {
            preparedStatement.setString(1,token);
            //W tym wypadku przekazujemy co ma byc pierwszym argumentem w zapytaniu DELETE_ENTRY_SQL które stworzyliśmy wyżej
            preparedStatement.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
    //Kod odpowiadający za usuwanie z bazy danych
    
    public List<Entry> getAllEntries() {
        List<Entry> entries = new ArrayList<>();
        try (Connection connection = DriverManager.getConnection(DB_URL);
             Statement statement = connection.createStatement();
             ResultSet resultSet = statement.executeQuery(SELECT_ENTRY_SQL)) {
            while (resultSet.next()) {
                String token = resultSet.getString("token");
                int x = resultSet.getInt("x");
                int y = resultSet.getInt("y");
                String color = resultSet.getString("color");
                entries.add(new Entry(token, x, y, color));
            }
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return entries;
    }
    //Publiczna lista zawierająca wszystkie wpisy z bazy danych
    
    
    public static class Entry {
        private String token;
        private int x;
        private int y;
        private String color;

        public Entry(String token, int x, int y, String color) {
            this.token = token;
            this.x = x;
            this.y = y;
            this.color = color;
        }

        public String getToken() { return token; }
        public int getX() { return x; }
        public int getY() { return y; }
        public String getColor() { return color; }
    }
    //Klasa pomocnicza moze się przydać może nie. Jak nie działa to lepiej mieć
}


```

</details>

<details>
  <summary>Przykładowe użycie bazy danych w aplikacji</summary>

```java

@RestController
public class ImageController {
    private static final int WIDTH = 512;
    private static final int HEIGHT = 512;

    private final BufferedImage image = new BufferedImage(WIDTH, HEIGHT, BufferedImage.TYPE_INT_RGB);
    //Inicjalizacja bazy danych
    private final DatabaseManager databaseManager;
    private final UserController userController;
    public ImageController(DatabaseManager databaseManager, UserController userController) {
        this.databaseManager = databaseManager;
        this.userController = userController;
        Graphics2D graphics = image.createGraphics();
        graphics.setColor(Color.black);
        graphics.fillRect(0, 0, WIDTH, HEIGHT);
        graphics.dispose();
    }

    @GetMapping("/image")
    public void getImage(HttpServletResponse response) throws IOException {
        response.setContentType("image/jpeg");
        ImageIO.write(image,"jpeg",response.getOutputStream());
    }

    @PostMapping("/pixel")
    public ResponseEntity<Void> putPixel(@RequestBody PixelRequest pixelRequest){
        String colorHex = pixelRequest.getColor().trim();
        int rgb = Integer.parseInt(colorHex, 16);
        image.setRGB(pixelRequest.getX(), pixelRequest.getY(),rgb);
        String timestamp = LocalDateTime.now().format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        //Wywolanie zapytania do bazy danych za pomocą wcześniej utworzonej metody insertEntry
        databaseManager.insertEntry(pixelRequest.getUuid().toString(), pixelRequest.getX(), pixelRequest.getY(), pixelRequest.getColor(), timestamp);
        return ResponseEntity.ok().build();
    }

}

```
  
</details>

## Połączenie serwer-klient
<details>

<summary>Utworzenie serwera do którego może się podłączyć tylko jeden użytkownik na przykładzie zadania o umieszczanie pixeli na obrazie a potem utworzenie klienta</summary>

```java

//Oznaczenie klasy jako @Component sprawia, że Spring
// automatycznie utworzy jej instancję podczas uruchamiania aplikacji.
@Component
public class TcpServer {
    private static final int PORT = 2136; // Port serwera TCP
    private static final String ADMIN_PASSWORD = "admin"; // hasło administratora
    private static final String DB_NAME = "jdbc:sqlite:database.db";
    private Connection database; // Dodanie połączenia do bazy danych
    private final UserController userController;

    //Podłączenie bazy danych
    @Autowired
    public TcpServer(UserController userController) {
        this.userController = userController;
    }

    //adnotacja @PostConstruct powoduje, że jest wykonywana natychmiast po utworzeniu instancji klasy.
    @PostConstruct
    public void startServer() {
        try {
            //Inicjalizacja połączenia z bazą danych
            database = DriverManager.getConnection(DB_NAME);
        // Serwer TCP uruchamia się w osobnym wątku, aby nie blokować głównego wątku aplikacji
        new Thread(() -> {
            try (ServerSocket serverSocket = new ServerSocket(PORT)) {
                System.out.println("Server started on port " + PORT);

                // Serwer akceptuje tylko jedno połączenie
                Socket clientSocket = serverSocket.accept();
                System.out.println("Client connected" + clientSocket.getInetAddress());

                // Utworzenie strumieni wejścia/wyjścia
                BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
                PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true);

                // Wysłanie zapytania do klienta o podanie hasła
                out.println("Input password: ");

                //Odczytanie hasła od klienta
                String receivedPassword = in.readLine();

                if (receivedPassword.equals(ADMIN_PASSWORD)) {
                    out.println("Password correct. Logged in as Admin");
                    System.out.println("Admin logged in");
                    //Tutaj znajduje się logika co może Administrator

                    // Pętla do obsługi poleceń
                    String command;
                    while ((command = in.readLine()) != null) {
                        if (command.startsWith("ban ")) {
                            String token = command.substring(4).trim();// Pobranie tokena z komendy
                            UUID uuid = UUID.fromString(token); // przekonwertowanie stringa na UUID
                            User user = findUser(uuid);
                            if (user != null) {
                                userController.users.remove(user);
                            }
                            int removedRecords = banUser(token);
                            out.println("Banned token " + token + "Deleted: " + removedRecords + " records");
                        } else {
                            out.println("Unknown command " + command);
                        }
                    }
                } else {
                    out.println("Password incorrect");
                    System.out.println("Incorrect password");
                    clientSocket.close(); //Zamykanie połączenia w przypadku błędnego hasła
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start(); // Uruchominie serwera w osobnym wątku.
    } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    // Metoda do usuwania użytkownika

    private int banUser(String token) {
        int removedRecord = 0;
        try {
            // Usuwanie rekordów z bazy danych
            String deleteSQL = "DELETE FROM entry WHERE token = ?";
            PreparedStatement preparedStatement = database.prepareStatement(deleteSQL);
            preparedStatement.setString(1, token);
            removedRecord = preparedStatement.executeUpdate();
        } catch (SQLException e) {
            e.printStackTrace();
        }
        return removedRecord;
    }

    //Znalezienie użytkownika na podstawie UUID

    private User findUser(UUID uuid) {
        return userController.users.stream()
                .filter(user -> user.getUuid().equals(uuid))
                .findFirst()
                .orElse(null);
    }
}

```

# Serwer

</details>
<details>
  <summary>Utworzenie serwera akceptującego wiele połączeń</summary>

Krok 1: Stwórz klasę ClientHandler, która będzie odpowiedzialna za obsługę pojedynczego klienta w osobnym wątku:

```java

import java.io.*;
import java.net.Socket;

public class ClientHandler implements Runnable {
    private final Socket clientSocket;
    private static final String ADMIN_PASSWORD = "supersecret"; // Hasło administratora

    public ClientHandler(Socket clientSocket) {
        this.clientSocket = clientSocket;
    }

    @Override
    public void run() {
        try {
            // Tworzenie strumieni wejścia/wyjścia
            BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
            PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true);

            // Wysłanie żądania hasła do klienta
            out.println("Podaj hasło:");

            // Odczytanie hasła od klienta
            String receivedPassword = in.readLine();

            // Sprawdzenie poprawności hasła
            if (ADMIN_PASSWORD.equals(receivedPassword)) {
                out.println("Hasło poprawne. Zalogowano jako Administrator.");
                System.out.println("Klient " + clientSocket.getInetAddress() + " zalogowany jako Administrator.");
                // Tutaj możesz dodać logikę działania serwera dla administratora
            } else {
                out.println("Niepoprawne hasło. Połączenie zostanie zakończone.");
                System.out.println("Błędne hasło od klienta " + clientSocket.getInetAddress() + ", rozłączanie.");
                clientSocket.close(); // Zamykanie połączenia w przypadku błędnego hasła
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                clientSocket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}

```

Krok 2: Zmodyfikuj klasę serwera
Dostosuj klasę serwera tak, aby akceptowała wiele połączeń i uruchamiała nowy wątek dla każdego klienta:

```java

import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;

@Component
public class TcpServer {
    private static final int PORT = 8081; // Port serwera TCP

    @PostConstruct
    public void startServer() {
        // Uruchom serwer TCP w osobnym wątku, aby nie blokować głównego wątku aplikacji Spring Boot
        new Thread(() -> {
            try (ServerSocket serverSocket = new ServerSocket(PORT)) {
                System.out.println("Serwer TCP uruchomiony i nasłuchuje na porcie " + PORT);

                while (true) { // Serwer działa w trybie nieskończonej pętli
                    // Serwer akceptuje nowe połączenie
                    Socket clientSocket = serverSocket.accept();
                    System.out.println("Połączono z klientem: " + clientSocket.getInetAddress());

                    // Tworzenie nowego wątku do obsługi klienta
                    ClientHandler clientHandler = new ClientHandler(clientSocket);
                    new Thread(clientHandler).start(); // Uruchomienie nowego wątku dla klienta
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start(); // Uruchamia serwer w osobnym wątku
    }
}

```

</details>

<details>
  <summary>Utworzenie klienta mogącego komunikować się z serwerem za pomocą komend</summary>

```java

public class Main {
    private static final String SERVER_ADDRESS = "localhost"; // Adres serwera
    private static final int SERVER_PORT = 2136; // Port serwera

    public static void main(String[] args) {
        try (Socket socket = new Socket(SERVER_ADDRESS, SERVER_PORT)) {
            BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
            BufferedReader consoleReader = new BufferedReader(new InputStreamReader(System.in));

            // Odczytaj i wyswietl wiadomość od serwera
            System.out.println(in.readLine());

            // Podaj hasło
            String password = consoleReader.readLine();
            out.println(password);

            // Odczytaj wynik weryfikacji hasła
            String response = in.readLine();
            System.out.println(response);
            if (response.equals("Password correct. Logged in as Admin")) {
                System.out.println("You can now type ban TOKEN to ban users.");
                //Utworzenie pętli aby móc wysyłać wiadomości do serwera
                while (true) {
                    String command = consoleReader.readLine();
                    out.println(command);

                    // Odczytaj odpowiedz od serwera
                    response = in.readLine();
                    System.out.println(response);

                    // Jesli komenda exit zakoncz dzialanie
                    if (command.equalsIgnoreCase("exit")) {
                        break;
                    }
                }
            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

```
  
</details>

## Tworzenie obrazu na stronie w Springu
<details>
  <summary>Proste utworzenie czarnego obrazu a potem wywołanie go na podstronie</summary>

```java
private static final int WIDTH = 512;
    private static final int HEIGHT = 512;

    private final BufferedImage image = new BufferedImage(WIDTH, HEIGHT, BufferedImage.TYPE_INT_RGB);
    //Inicjalizacja bazy danych
    private final DatabaseManager databaseManager;
    private final UserController userController;
    public ImageController(DatabaseManager databaseManager, UserController userController) {
        this.databaseManager = databaseManager;
        this.userController = userController;
        Graphics2D graphics = image.createGraphics();
        graphics.setColor(Color.black);
        graphics.fillRect(0, 0, WIDTH, HEIGHT);
        graphics.dispose();
    }

    @GetMapping("/image")
    public void getImage(HttpServletResponse response) throws IOException {
        response.setContentType("image/jpeg");
        ImageIO.write(image,"jpeg",response.getOutputStream());
    }
```
  
</details>

## Pliki CSV
<details>
    <summary>Odczytywanie danych z pliku CSV za pomocą Apache Commons CSV</summary>

### Krok 1: Dodanie zależności

Dodaj następującą zależność do pliku `pom.xml`:

```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-csv</artifactId>
    <version>1.8</version>
</dependency>
```

### Krok 2: Struktura pliku CSV

Upewnij się, że Twój plik CSV ma odpowiednią strukturę. Przykład pliku `events.csv`:

```csv
Name,Time
Event 1,12:00:00
Event 2,14:30:15
Event 3,09:45:00
```

### Krok 3: Klasa modelu danych

Stwórz klasę, która będzie reprezentować dane z pliku CSV. W tym przypadku stworzymy klasę `Event`.

```java
import java.time.LocalTime;

public class Event {
    private String name;
    private LocalTime time;

    public Event(String name, LocalTime time) {
        this.name = name;
        this.time = time;
    }

    public String getName() {
        return name;
    }

    public LocalTime getTime() {
        return time;
    }

    @Override
    public String toString() {
        return "Event{name='" + name + "', time=" + time + '}';
    }
}
```

### Krok 4: Odczytywanie danych z pliku CSV

Stwórz klasę, która będzie odpowiedzialna za odczyt danych z pliku CSV i ich przetwarzanie.

```java
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVRecord;

import java.io.FileReader;
import java.io.IOException;
import java.nio.file.Paths;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.List;

public class CsvReader {
    public static List<Event> readEventsFromCsv(String filePath) {
        List<Event> events = new ArrayList<>();
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("HH:mm:ss");

        try (FileReader reader = new FileReader(filePath);
             CSVParser csvParser = new CSVParser(reader, CSVFormat.DEFAULT.withHeader())) {

            for (CSVRecord csvRecord : csvParser) {
                String name = csvRecord.get("Name");
                String timeString = csvRecord.get("Time");
                LocalTime time = LocalTime.parse(timeString, formatter);

                events.add(new Event(name, time));
            }

        } catch (IOException e) {
            e.printStackTrace();
        }

        return events;
    }

    public static void main(String[] args) {
        String filePath = "events.csv";
        List<Event> events = readEventsFromCsv(filePath);

        for (Event event : events) {
            System.out.println(event);
        }
    }
}
```
</details>

<details>
    <summary>Odczytywanie godziny w formacie HH:mm:ss z pliku CSV</summary>

### Krok 1: Struktura pliku CSV

Twój plik CSV (`events.csv`) powinien mieć kolumny `Name` i `Time`. Przykład zawartości pliku:

```csv
Name,Time
Event 1,12:00:00
Event 2,14:30:15
Event 3,09:45:00
```

### Krok 2: Klasa modelu danych


```java
import java.time.LocalTime;

public class Event {
    private String name;
    private LocalTime time;

    public Event(String name, LocalTime time) {
        this.name = name;
        this.time = time;
    }

    public String getName() {
        return name;
    }

    public LocalTime getTime() {
        return time;
    }

    @Override
    public String toString() {
        return "Event{name='" + name + "', time=" + time + '}';
    }
}
```

### Krok 3: Moduł do odczytywania danych z pliku CSV

Upewnij się, że masz moduł do odczytywania danych z pliku CSV.

```java
import org.apache.commons.csv.CSVFormat;
import org.apache.commons.csv.CSVParser;
import org.apache.commons.csv.CSVRecord;

import java.io.FileReader;
import java.io.IOException;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;
import java.util.ArrayList;
import java.util.List;

public class CsvReader {
    public static List<Event> readEventsFromCsv(String filePath) {
        List<Event> events = new ArrayList<>();
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("HH:mm:ss");

        try (FileReader reader = new FileReader(filePath);
             CSVParser csvParser = new CSVParser(reader, CSVFormat.DEFAULT.withHeader())) {

            for (CSVRecord csvRecord : csvParser) {
                String name = csvRecord.get("Name");
                String timeString = csvRecord.get("Time");
                LocalTime time = LocalTime.parse(timeString, formatter);

                events.add(new Event(name, time));
            }

        } catch (IOException e) {
            e.printStackTrace();
        }

        return events;
    }
}
```

### Krok 4: Użycie modułu do odczytu godzin z pliku CSV

Stwórz klasę, która będzie używać modułu do odczytywania danych z pliku CSV i wyświetlania godzin.

```java

import java.util.List;

public class Main {
    public static void main(String[] args) {
        String filePath = "events.csv";
        List<Event> events = CsvReader.readEventsFromCsv(filePath);

        for (Event event : events) {
            System.out.println("Wydarzenie: " + event.getName() + ", Godzina: " + event.getTime());
        }
    }
}

```

</details>

## JavaFX
WAŻNE: aby uniknąć błędów przy tworzeniu aplikacji z javafx i nie bawić się w szukanie repo należy w inteliJ utworzyć nowy projekt i wybrać javaFX.

Każda aplikacja z javaFX powinna zacząć się od zainicjalizowania pliku view.fxml oraz wystartowania aplikacji
<details>
    <summary>Main aplikacji</summary>
    
```java
    
public class HelloApplication extends Application {
    @Override
    public void start(Stage stage) throws Exception {
        FXMLLoader fxmlLoader = new FXMLLoader(getClass().getResource("view.fxml"));
        //Załadowanie pliku view.fxml
        stage.setScene(new Scene(fxmlLoader.load()));
        //Utworzenie sceny
        stage.setTitle("Hello World");
        stage.show();
        //Wyswietlenie aplikacji
    }
    public static void main(String[] args) {
        launch(args);
    }
}
```
</details>

Jeśli będzie trzeba stworzyć aplikacje to prawdpodobnie będzie podany plik view.fxml
w pliku view.fxml pola odczytywalne/modyfikowalne oznaczane są fx:id tak jak tutaj:
```
        <TextField fx:id="filterField" />
        <ListView fx:id="wordList" VBox.vgrow="ALWAYS" />
        <HBox spacing="10.0">
            <children>
                <Label text="Total words:" />
                <Label fx:id="wordCountLabel" alignment="CENTER_RIGHT" text="0" />
            </children>
        </HBox>
    </children>
</VBox>
```
<details>
    <summary>Aplikacja w javaFX wykorzystująca podaną wcześniej klasę main na podstawie kolokwium z 2022</summary>
    
```java
public class HelloController {
    //Adnotacja @FXML do każdej zmiennej z view.fxml
    @FXML
    private ListView<String> wordList; // Komponent ListView w interfejsie użytkownika,
    // który wyświetla listę słów otrzymanych z serwera

    @FXML
    private Label wordCountLabel; // Komponent Label, który wyświetla ilość słów (lub inne informacje),
    // może być aktualizowany na bieżąco

    @FXML
    private TextField filterField; // Pole tekstowe, w którym użytkownik wpisuje filtr,
    // na podstawie którego lista słów jest filtrowana

    private List<String> allWords = new ArrayList<>();
    //Lista z wszystkimi słowami
    private final SimpleDateFormat timeFormat = new SimpleDateFormat("HH:mm:ss");
    //Stworzenie prostego formatu czasowego aby wyświetlić czas w formacie HH:mm:ss

    public void initialize() {
        // Dodajemy nasłuchiwacza (listener) na zmianę tekstu w polu tekstowym 'filterField'
        // Kiedy użytkownik wpisze coś nowego w polu tekstowym, metoda 'filterWords' zostanie wywołana z nową wartością filtra (newValue)
        filterField.textProperty().addListener((observable, oldValue, newValue) -> filterWords(newValue));

        // Inicjujemy połączenie z serwerem - metoda 'connectToServer' uruchamia wątek, który nawiązuje połączenie i odbiera dane
        connectToServer();
    }
    //inicjalizacja aplikacji
    private void connectToServer() {
        new Thread(() -> {
            //Utworzenie nowego wątku
            try (Socket socket = new Socket("localhost", 5000);
                 //podłączenie do serwera na porcie 5000
                 BufferedReader reader = new BufferedReader(new InputStreamReader(socket.getInputStream()))) {

                String word;
                while ((word = reader.readLine()) != null) {
                    //Pętla do odbierania danych
                    String timestampedWord = timeFormat.format(new Date()) + " " + word;
                    allWords.add(timestampedWord);
                    //Każda linia danych (word) otrzymuje znacznik czasu (w formacie daty i czasu).
                    // Ta linia zostaje dodana do listy allWords.
                    // Ponieważ modyfikowanie elementów interfejsu użytkownika
                    // musi odbywać się w głównym wątku aplikacji (np. w przypadku aplikacji JavaFX)
                    Platform.runLater(() -> updateWordList());
                    //Platform.runLater() używana jest do wywołania metody updateWordList()
                    // , która zaktualizuje widok listy słów.
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }).start();
    }

    private void updateWordList() {
        // Aktualizacja licznika
        wordCountLabel.setText(String.valueOf(allWords.size()));

        // Filtrowanie i sortowanie
        filterWords(filterField.getText());
    }

    private void filterWords(String filter) {
        // Tworzymy nową listę, która będzie przechowywać przefiltrowane słowa
        List<String> filteredWords = new ArrayList<>();

        // Przechodzimy przez wszystkie słowa z listy allWords
        for (String word : allWords) {
            // Rozdzielamy tekst na dwie części: czas i faktyczne słowo
            String[] parts = word.split(" ", 2);
            String actualWord = parts[1]; // Słowo po czasie

            // Sprawdzamy, czy filtr jest pusty, lub czy słowo zaczyna się od podanego filtru
            if (filter.isEmpty() || actualWord.startsWith(filter)) {
                // Jeśli spełnia warunki, dodajemy słowo do listy przefiltrowanych
                filteredWords.add(word);
            }
        }

        // Sortujemy listę przefiltrowanych słów alfabetycznie względem faktycznych słów (ignorując część z czasem)
        Collections.sort(filteredWords, (a, b) -> a.split(" ", 2)[1].compareTo(b.split(" ", 2)[1]));

        // Aktualizujemy widok listy słów w interfejsie użytkownika, ustawiając nową, przefiltrowaną listę
        wordList.getItems().setAll(filteredWords);
    }
}
```
</details>

## Niedziałający program z javafx ale kod może się przydać w celu tworzenia okienka w kodzie

<details>
    <summary>kod</summary>

```java
    public class ImageServer {

    private static final int PORT = 12345; // Port na którym serwer będzie nasłuchiwał
    private static final String IMAGE_DIR = "images"; // Katalog na obrazy
    private static final String DB_FILE = IMAGE_DIR + "/index.db"; // Ścieżka do pliku bazy danych
    private static int filterSize = 3;  // Rozmiar filtra (domyślnie 3)

    public static void main(String[] args) {
        createImageDirectory(); // Tworzy katalog na obrazy, jeśli nie istnieje
        setupDatabase(); // Inicjalizuje bazę danych
        startServer(); // Rozpoczyna działanie serwera
    }

    private static void createImageDirectory() {
        File dir = new File(IMAGE_DIR); // Tworzy obiekt File dla katalogu
        if (!dir.exists()) { // Sprawdza, czy katalog istnieje
            dir.mkdirs(); // Tworzy katalog, jeśli nie istnieje
        }
    }

    private static void setupDatabase() {
        try (Connection conn = DriverManager.getConnection("jdbc:sqlite:" + DB_FILE)) { // Nawiązuje połączenie z bazą danych SQLite
            Statement stmt = conn.createStatement(); // Tworzy obiekt Statement do wykonywania zapytań
            String createTableSQL = "CREATE TABLE IF NOT EXISTS images (" + // SQL do tworzenia tabeli, jeśli nie istnieje
                    "id INTEGER PRIMARY KEY AUTOINCREMENT," + // Kolumna id jako klucz główny z autoinkrementacją
                    "path TEXT NOT NULL," + // Kolumna path (ścieżka do pliku) jako tekst, nie może być NULL
                    "size INTEGER NOT NULL," + // Kolumna size (rozmiar filtra) jako integer, nie może być NULL
                    "delay INTEGER NOT NULL);"; // Kolumna delay (czas działania algorytmu) jako integer, nie może być NULL
            stmt.execute(createTableSQL); // Wykonuje zapytanie SQL do utworzenia tabeli
        } catch (SQLException e) { // Obsługuje wyjątek SQL
            e.printStackTrace(); // Wypisuje stos śladu błędu
        }
    }

    private static void startServer() {
        ExecutorService executor = Executors.newSingleThreadExecutor(); // Tworzy ExecutorService do obsługi wątków (jednowątkowo)
        try (ServerSocket serverSocket = new ServerSocket(PORT)) { // Tworzy obiekt ServerSocket do nasłuchiwania połączeń na porcie
            System.out.println("Server started on port " + PORT); // Wypisuje informację o uruchomieniu serwera
            while (true) { // Pętla nieskończona do obsługi połączeń
                Socket clientSocket = serverSocket.accept(); // Akceptuje połączenie klienta
                executor.submit(() -> handleClient(clientSocket)); // Wysyła połączenie do obsługi w osobnym wątku
            }
        } catch (IOException e) { // Obsługuje wyjątek wejścia/wyjścia
            e.printStackTrace(); // Wypisuje stos śladu błędu
        }
    }

    private static void handleClient(Socket clientSocket) {
        try (InputStream input = clientSocket.getInputStream(); // Otwiera strumień wejściowy z klienta
             OutputStream output = clientSocket.getOutputStream()) { // Otwiera strumień wyjściowy do klienta

            // 1. Receive and save PNG file
            String fileName = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd_HH-mm-ss")) + ".png"; // Generuje nazwę pliku z aktualnym znakiem czasowym
            File file = new File(IMAGE_DIR, fileName); // Tworzy obiekt File dla nowego pliku
            try (FileOutputStream fos = new FileOutputStream(file)) { // Otwiera strumień wyjściowy do zapisu pliku
                byte[] buffer = new byte[4096]; // Bufor do przechwytywania danych
                int bytesRead;
                while ((bytesRead = input.read(buffer)) != -1) { // Odczytuje dane od klienta
                    fos.write(buffer, 0, bytesRead); // Zapisuje dane do pliku
                }
            }
            
            filterSize = showFilterSizeUI(); // Wyświetla UI do wyboru rozmiaru filtra
            
            BufferedImage image = ImageIO.read(file); // Odczytuje obraz z pliku
            long startTime = System.currentTimeMillis(); // Zapisuje czas rozpoczęcia
            BufferedImage blurredImage = applyBoxBlur(image, filterSize); // Zastosowuje filtr box blur
            long endTime = System.currentTimeMillis(); // Zapisuje czas zakończenia
            long delay = endTime - startTime; // Oblicza czas działania algorytmu
            
            try (Connection conn = DriverManager.getConnection("jdbc:sqlite:" + DB_FILE)) { // Nawiązuje połączenie z bazą danych SQLite
                String insertSQL = "INSERT INTO images (path, size, delay) VALUES (?, ?, ?)"; // SQL do wstawienia danych
                try (PreparedStatement pstmt = conn.prepareStatement(insertSQL)) { // Przygotowuje zapytanie do bazy danych
                    pstmt.setString(1, file.getAbsolutePath()); // Ustawia ścieżkę pliku
                    pstmt.setInt(2, filterSize); // Ustawia rozmiar filtra
                    pstmt.setLong(3, delay); // Ustawia czas działania algorytmu
                    pstmt.executeUpdate(); // Wykonuje zapytanie
                }
            }
            
            ImageIO.write(blurredImage, "png", output); // Zapisuje przetworzony obraz do strumienia wyjściowego

        } catch (IOException | SQLException e) { // Obsługuje wyjątki wejścia/wyjścia i SQL
            e.printStackTrace(); // Wypisuje stos śladu błędu
        }
    }

    private static BufferedImage applyBoxBlur(BufferedImage image, int size) {
        // Box blur implementation
        int radius = size / 2; // Oblicza promień filtra
        int width = image.getWidth(); // Pobiera szerokość obrazu
        int height = image.getHeight(); // Pobiera wysokość obrazu
        BufferedImage blurredImage = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB); // Tworzy nowy obraz dla przetworzonego obrazu
        for (int x = 0; x < width; x++) { // Iteruje przez szerokość obrazu
            for (int y = 0; y < height; y++) { // Iteruje przez wysokość obrazu
                int r = 0, g = 0, b = 0, count = 0; // Inicjalizuje zmienne do obliczenia średnich wartości RGB
                for (int dx = -radius; dx <= radius; dx++) { // Iteruje przez sąsiednie piksele w poziomie
                    for (int dy = -radius; dy <= radius; dy++) { // Iteruje przez sąsiednie piksele w pionie
                        int nx = x + dx; // Oblicza współrzędne sąsiada w poziomie
                        int ny = y + dy; // Oblicza współrzędne sąsiada w pionie
                        if (nx >= 0 && nx < width && ny >= 0 && ny < height) { // Sprawdza, czy sąsiad mieści się w granicach obrazu
                            Color color = new Color(image.getRGB(nx, ny)); // Pobiera kolor piksela sąsiada
                            r += color.getRed(); // Dodaje wartość czerwieni
                            g += color.getGreen(); // Dodaje wartość zieleni
                            b += color.getBlue(); // Dodaje wartość niebieskiego
                            count++; // Inkrementuje licznik
                        }
                    }
                }
                Color avgColor = new Color(r / count, g / count, b / count); // Oblicza średni kolor
                blurredImage.setRGB(x, y, avgColor.getRGB()); // Ustawia średni kolor w przetworzonym obrazie
            }
        }
        return blurredImage; // Zwraca przetworzony obraz
    }

    private static int showFilterSizeUI() {
        final int[] selectedSize = {3}; // Tablica do przechowywania wybranego rozmiaru filtra

        Application.launch(FilterSizeApp.class, (String[]) null); // Uruchamia aplikację JavaFX do wyboru rozmiaru filtra

        return filterSize; // Zwraca wybrany rozmiar filtra
    }

    public static class FilterSizeApp extends Application {
        @Override
        public void start(Stage primaryStage) {
            primaryStage.setTitle("Select Filter Size"); // Ustawia tytuł okna

            Slider slider = new Slider(1, 15, 3); // Tworzy suwak do wyboru rozmiaru filtra (zakres 1-15, domyślnie 3)
            slider.setMajorTickUnit(2); // Ustawia jednostki dużych kroków suwaka
            slider.setShowTickLabels(true); // Pokazuje etykiety przy dużych krokach
            slider.setShowTickMarks(true); // Pokazuje oznaczenia suwaka

            Label label = new Label("Filter Size: " + (int) slider.getValue()); // Etykieta pokazująca rozmiar filtra
            slider.valueProperty().addListener((obs, oldVal, newVal) -> label.setText("Filter Size: " + newVal.intValue())); // Aktualizuje etykietę, gdy zmienia się wartość suwaka

            Button submitButton = new Button("Submit"); // Tworzy przycisk do zatwierdzania wyboru
            submitButton.setOnAction(e -> {
                filterSize = (int) slider.getValue(); // Ustawia wybrany rozmiar filtra
                primaryStage.close(); // Zamknięcie okna
            });

            VBox vbox = new VBox(10, slider, label, submitButton); // Tworzy kontener dla elementów GUI
            Scene scene = new Scene(vbox, 300, 200); // Tworzy scenę z kontenerem
            primaryStage.setScene(scene); // Ustawia scenę w oknie
            primaryStage.showAndWait(); // Wyświetla okno i czeka na jego zamknięcie
        }
    }
}
```   
</details>
