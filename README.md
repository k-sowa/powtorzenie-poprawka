
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

<summary>Utworzenie serwera na przykładzie zadania o umieszczanie pixeli na obrazie a potem utworzenie klienta</summary>

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
