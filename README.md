# Sunflower_Model

이전 챕터에서 ViewModel에 대해 보았는데 이어서 MVVM에서 서버 or 로컬 DB와의 통신을 담당하는 Data Layer인 Repository 클래스를 살펴보려고 합니다.

## PlantRepository
![image](https://user-images.githubusercontent.com/48902047/149607220-116a3cd2-6c82-41a3-b5b0-5d7a9e9a1add.png)

 ```Kotlin
@Singleton
class PlantRepository @Inject constructor(private val plantDao: PlantDao) {

    fun getPlants() = plantDao.getPlants()

    fun getPlant(plantId: String) = plantDao.getPlant(plantId)

    fun getPlantsWithGrowZoneNumber(growZoneNumber: Int) =
        plantDao.getPlantsWithGrowZoneNumber(growZoneNumber)
}
```
Dao로 부터 데이터를 불러오는 함수가 구현되어 있습니다.

Room의 구성요소인 Entity, Dao, RoomDatabase도 한번 살펴보겠습니다.

## Plant(Entity)

 ```Kotlin
@Entity(tableName = "plants")
data class Plant(
    @PrimaryKey @ColumnInfo(name = "id") val plantId: String,
    val name: String,
    val description: String,
    val growZoneNumber: Int,
    val wateringInterval: Int = 7, // how often the plant should be watered, in days
    val imageUrl: String = ""
) {

    /**
     * Determines if the plant should be watered.  Returns true if [since]'s date > date of last
     * watering + watering Interval; false otherwise.
     */
    fun shouldBeWatered(since: Calendar, lastWateringDate: Calendar) =
        since > lastWateringDate.apply { add(DAY_OF_YEAR, wateringInterval) }

    override fun toString() = name
}
```
먼저 @Entity + tableName 속성으로 테이블 이름이 "plants"로 설정되어 있습니다. 그리고 plantId 가 테이블상 이름으로는 @ColumInfo 로 "id" 라는 이름을 갖게되고 @PrimaryKey로 고유키가 됩니다.
## PlantDao(DAO)
 ```Kotlin
@Dao
interface PlantDao {
    @Query("SELECT * FROM plants ORDER BY name")
    fun getPlants(): Flow<List<Plant>>

    @Query("SELECT * FROM plants WHERE growZoneNumber = :growZoneNumber ORDER BY name")
    fun getPlantsWithGrowZoneNumber(growZoneNumber: Int): Flow<List<Plant>>

    @Query("SELECT * FROM plants WHERE id = :plantId")
    fun getPlant(plantId: String): Flow<Plant>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(plants: List<Plant>)
}
```
@Dao 로 Dao 인터페이스임을 알려줍니다. 그리고 @Query 를 사용하여 쿼리문을 작성할 수 있고 함수의 파라미터는 쿼리문에서 ' : ' 를 앞에 붙임으로써 사용할 수 있습니다. 그리고 @Insert와 onConfilct 충돌전략 속성을 사용하여 데이터를 데이터베이스에 쉽게 저장할 수도 있습니다.

그리고 코루틴에서 실행될 함수는 suspend 키워드가 함수 앞에 붙어 있는것을 확인 할 수 있습니다.

## AppDatabase(RoomDatabase)
 ```Kotlin
/**
 * The Room database for this app
 */
@Database(entities = [GardenPlanting::class, Plant::class], version = 1, exportSchema = false)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    abstract fun gardenPlantingDao(): GardenPlantingDao
    abstract fun plantDao(): PlantDao

    companion object {

        // For Singleton instantiation
        @Volatile private var instance: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return instance ?: synchronized(this) {
                instance ?: buildDatabase(context).also { instance = it }
            }
        }

        // Create and pre-populate the database. See this article for more details:
        // https://medium.com/google-developers/7-pro-tips-for-room-fbadea4bfbd1#4785
        private fun buildDatabase(context: Context): AppDatabase {
            return Room.databaseBuilder(context, AppDatabase::class.java, DATABASE_NAME)
                .addCallback(
                    object : RoomDatabase.Callback() {
                        override fun onCreate(db: SupportSQLiteDatabase) {
                            super.onCreate(db)
                            val request = OneTimeWorkRequestBuilder<SeedDatabaseWorker>().build()
                            WorkManager.getInstance(context).enqueue(request)
                        }
                    }
                )
                .build()
        }
    }
}
```
RoomDatabase는 RoomDatabase()를 상속받아야한다. @Database 에는 테이블을 담당하는 Entity들과 데이터베이스 관련 속성들이 들어가 있습니다. 그리고 @TypeConverter도 있는데 테이블에 저장할 값 타입을 변환하거나 다른 형태로 변환할 경우 사용합니다. (ex. 데이터베이스에는 List타입을 저장못해 json형식으로 변환해야합니다.)

그리고 데이터베이스가 계속 생성되어 메모리가 낭비되는 것을 방지하기 위해 @Volatule과 getInstance() 함수로 싱글톤으로 데이터베이스가 생성되게 되어있습니다.

마지막으로 buildDatabase()를 살펴보면 RoomDatabase를 빌드(생성)하는 건데 데이터베이스의 이름을 설정할 수 있습니다. 그리고 여기서는 RoomDatabase callback을 사용해서 데이터베이스가 생성되면 한번만 실행되는 OneTimeWorkRequest 형식의 WorkManager 컴포넌트를 사용하여 데이터베이스에 초기값을 넣는 로직이 실행됩니다.

 ## Converters
  ```Kotlin
/**
 * Type converters to allow Room to reference complex data types.
 */
class Converters {
    @TypeConverter fun calendarToDatestamp(calendar: Calendar): Long = calendar.timeInMillis

    @TypeConverter fun datestampToCalendar(value: Long): Calendar =
        Calendar.getInstance().apply { timeInMillis = value }
}
```
TypeConverter는 위와 같이 구현되어 있다. 타임스탬프를 날짜로 변환하는데 사용합니다.

## GardenPlantingRepository
![image](https://user-images.githubusercontent.com/48902047/149607432-f6536d0c-14d7-4419-93cf-f20c3a0053ea.png)
  ```Kotlin
@Singleton
class GardenPlantingRepository @Inject constructor(
        private val gardenPlantingDao: GardenPlantingDao
) {

    suspend fun createGardenPlanting(plantId: String) {
        val gardenPlanting = GardenPlanting(plantId)
        gardenPlantingDao.insertGardenPlanting(gardenPlanting)
    }

    suspend fun removeGardenPlanting(gardenPlanting: GardenPlanting) {
        gardenPlantingDao.deleteGardenPlanting(gardenPlanting)
    }

    fun isPlanted(plantId: String) =
            gardenPlantingDao.isPlanted(plantId)

    fun getPlantedGardens() = gardenPlantingDao.getPlantedGardens()
}
```
위에서 본 것과 똑같이 Dao 인터페이스의 함수를 호출하고 있다. 그리고 코루틴에서 실행될 함수 앞에 suspend가 붙어 있는것을 확인할 수 있습니다.

이와 관련된 Entity 클래스와 Dao 인터페이스를 살펴보겠습니다.

## GardenPlanting(Entity)

  ```Kotlin
/**
 * [GardenPlanting] represents when a user adds a [Plant] to their garden, with useful metadata.
 * Properties such as [lastWateringDate] are used for notifications (such as when to water the
 * plant).
 *
 * Declaring the column info allows for the renaming of variables without implementing a
 * database migration, as the column name would not change.
 */
@Entity(
    tableName = "garden_plantings",
    foreignKeys = [
        ForeignKey(entity = Plant::class, parentColumns = ["id"], childColumns = ["plant_id"])
    ],
    indices = [Index("plant_id")]
)
data class GardenPlanting(
    @ColumnInfo(name = "plant_id") val plantId: String,

    /**
     * Indicates when the [Plant] was planted. Used for showing notification when it's time
     * to harvest the plant.
     */
    @ColumnInfo(name = "plant_date") val plantDate: Calendar = Calendar.getInstance(),

    /**
     * Indicates when the [Plant] was last watered. Used for showing notification when it's
     * time to water the plant.
     */
    @ColumnInfo(name = "last_watering_date")
    val lastWateringDate: Calendar = Calendar.getInstance()
) {
    @PrimaryKey(autoGenerate = true)
    @ColumnInfo(name = "id")
    var gardenPlantingId: Long = 0
}
```
처음 본 Entity와 다른점이 몇가지가 있는데 먼저 foreignKeys 즉 외래키 제약이 설정되어 있습니다.

외부키 제약을 맺음으로서 참조한 엔티티가 업데이트 될때 어떤일이 발생했는지 명시하는것을 허용하고 있습니다. @ForeignKey(onDelete=CASCADE) 어노테이션을 가지고 있는 어떤 Plant(parent) 객체가 삭제된다면 모든 해당 Plant의 모든 GardenPlanting(child)을 삭제하라는 명령을 SQLite에게 줄수도 있습니다.

Plant라는 Entity와 관계를 맺을거고 parentColums는 상위 항목의 기본 키 열 이름으로 설정(Plant Entity의 id)하고 childColumns에는 현재의 Entity인 GardenPlanting의 plant_id 키를 설정합니다.

이때 parentColums와 childColumns 개수는 동일해야합니다.

indicies는 인덱싱을 해서 SELECT 쿼리의 속도를 높여주는 역할을 한다. plant_id 로 인덱싱을 설정하였습니다. 만약 인덱스를 따로 설정하지 않으면  Room에서 index_${tableName} 로 기본값을 설정해놓습니다.

## GardenPlantingDao(Dao)
  ```Kotlin
@Dao
interface GardenPlantingDao {
    @Query("SELECT * FROM garden_plantings")
    fun getGardenPlantings(): Flow<List<GardenPlanting>>

    @Query("SELECT EXISTS(SELECT 1 FROM garden_plantings WHERE plant_id = :plantId LIMIT 1)")
    fun isPlanted(plantId: String): Flow<Boolean>

    /**
     * This query will tell Room to query both the [Plant] and [GardenPlanting] tables and handle
     * the object mapping.
     */
    @Transaction
    @Query("SELECT * FROM plants WHERE id IN (SELECT DISTINCT(plant_id) FROM garden_plantings)")
    fun getPlantedGardens(): Flow<List<PlantAndGardenPlantings>>

    @Insert
    suspend fun insertGardenPlanting(gardenPlanting: GardenPlanting): Long

    @Delete
    suspend fun deleteGardenPlanting(gardenPlanting: GardenPlanting)
}
```
@Transaction 을 활용하여 메서드의 질의들이 하나의 트랜잭션 안에서 실행하게 구현되어있습니다. 여기서는 두 개의 테이블에서 쿼리하는데 그것들을 하나의 트랜잭션에서 실행시키게 됩니다. 트랜잭션은 메서드의 Body 내부에서 Exception이 발생하면 반영되지않습니다.

@Delete는 Primary Key 로 매칭하여 알아서 해당 객체를 데이터베이스에서 삭제해줍니다.
