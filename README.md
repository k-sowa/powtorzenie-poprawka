# powtorzenie-poprawka


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
