## Connection Pool은 무엇일까요? 

 Connection? 연결? 어떤 연결을 의미할까 생각하실 수 있을 것입니다. 여기서 말하는 Connection이란 WAS와 데이터베이스 사이의 연결을 의미합니다. 해당 클라이언트와 서버 사이의 연결을 위해서는 아래 그림과 같이 3-way-handshaking 이라는 작업이 필요합니다.  



![three_handshaking](https://user-images.githubusercontent.com/54772162/103903326-1dd1a480-513f-11eb-85f4-910a857b5da1.PNG)

 3-way-handshaking은 3번의 패킷 교환을 통해 소켓을 형성하고 통신을 준비하는 과정을 의미하는데 이 과정이 쿼리를 요청할 때마다 반복적으로 실행된다면 이 또한 네트워크 구간에서 병목의 원인이 될 수 있습니다.

 

 실제 데이터베이스가 쿼리를 요청받고, 수행하는 과정을 간단하게 MySQL 공식문서의 내용을 참고하여 알아보겠습니다.



 아래 내용은 MySQL 8.0 기준으로 `INSERT`문을 수행하는 과정을 나타냅니다. 괄호 안의 숫자는 각 과정을 수행하는 데 필요한 비용의 비율을 의미합니다. 자세하게 확인하고 싶으신 분들은 아래 링크를 통해 참고 부탁드립니다.

> https://dev.mysql.com/doc/refman/8.0/en/insert-optimization.html

> - Connecting: (3)
> - Sending query to server: (2)
> - Parsing query: (2)
> - Inserting row: (1)
> - Inserting index: (1)
> - Closing: (1)

 가장 비용이 많이 드는 작업이 보이시나요? 



 바로 서버가 DB 접속하기 위해서 Connection을 생성하는 작업이 가장 큰 비용을 차지합니다. 즉, Connection을 반복적으로 생성하는 것은 그만큼 큰 비용이 드는 작업임을 알 수 있습니다.  



 이를 해결하기 위해서 사용하는 것이 Connection Pool 방식입니다. Connection Pool 방식은 매번 소켓을 생성하는 것이 아니라 미리 정해진 개수의 Connection을 생성해서 Pool에 보관하다가 재사용하여 데이터를 교환하는 방식입니다. 이러한 방식은 이미 형성된 Connection을 재사용한다는 점에서 반복적인 3-way-handshaking 과정을 제거할 수 있으므로 훨씬 더 빠르게 통신할 수 있습니다.



 지금까지 Connection 생성은 비싼 작업이라는 것을 알았고, 이를 위해 Connection Pool 방식을 사용한다고 알게 되었습니다.



그렇다면 지금부터 Connection Pool이 어떻게 동작하는지 알아보겠습니다.



## Connection Pool은 어떻게 동작할까?

  Spring의 default JDBC Connection Pool인 Hikari CP가 동작하는 방식을 통해서 Connection Pool이 동작하는 원리에 대해 간단히 알아보겠습니다. (해당 내용은 제가 이해한 내용을 알기 쉽게 표현한 것이라 구체적인 구현체 이름이나 객체 이름을 사용하지 않았습니다. 더 자세히 알고싶으신 분은 아래 첨부한 링크를 활용해주시길 바랍니다.)

> 우아한 형제들 블로그 : https://woowabros.github.io/experience/2020/02/06/hikaricp-avoid-dead-lock.html

![HikariCP_logic_success](https://user-images.githubusercontent.com/54772162/103903409-3215a180-513f-11eb-9146-dcbc39bc3a18.PNG)



 그림에서 볼 수 있듯이 Thread가 Connection을 요청하면 Connection Pool의 각자의 방식에 따라 유휴 Connection을 찾아서 반환합니다. Hikari CP의 경우, 이전에 사용했던 Connection이 존재하는지 확인하고, 이를 우선적으로 반환하는 특징을 갖고 있습니다. 

![wait_connection](https://user-images.githubusercontent.com/54772162/103903459-422d8100-513f-11eb-9cf8-58b2d3c036f8.PNG)

 만약에 가능한 Connection이 존재하지 않으면 HandOffQueue를 Polling하면서 다른 Thread가 Connection을 반납하기를 기다립니다. 이를 지정한 TimeOut 시간까지 대기하다가 시간이 만료되면 Exception을 던집니다.

![return_connection](https://user-images.githubusercontent.com/54772162/103903516-55d8e780-513f-11eb-8fb7-596cda4c8163.PNG)

 최종적으로 사용한 Connection을 반납하면 Connection Pool이 사용내역을 Connection 사용 내역을 기록하고, 반납된 Connection에 반납된 Connection을 삽입합니다. 이를 통해 HandOffQueue를 Polling 하던 Thread는 Connection을 획득하고 작업을 이어나갈 수 있게 됩니다.



 이렇게 Thread들은 트랜잭션을 처리하기 위해서 Connection을 획득하고, 이를 반납하므로써 반복적인 Connection 생성을 줄일 수 있었습니다.