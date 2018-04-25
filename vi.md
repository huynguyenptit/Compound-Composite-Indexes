[Nguồn](https://mariadb.com/kb/en/library/compound-composite-indexes/ "Liên kết với các chỉ mục tổng hợp (Composite) - MariaDB Knowledge Base")

# Các INDEX tổng hợp (Composite) - Cơ sở kiến thức về MariaDB

## Một bài tập nhỏ trong "Các INDEX tổng hợp" ("composite indexes")

Tài liệu này bắt đầu từ những thứ không đáng kể và có lẽ là buồn tẻ, nhưng khi xây dựng chúng lên sẽ có nhiều thông tin rất thú vị, có lẽ nhiều điều bạn đã không nhận ra về cách đánh dấu index của MariaDB vs MySQL.

Điều này cũng giải thích [Giải thích][1] (ở một mức độ nào đó)

(Hầu hết những điều này đều áp dụng cho các cơ sở dữ liệu non-MySQL).

## Câu truy vấn để thảo luận

Câu hỏi là "Andrew Johnson đã trở thành tổng thống của nước Mỹ khi nào?".

Bảng "Presidents" có dạng như sau:
    
    +-----+------------+----------------+-----------+
    | seq | last_name  | first_name     | term      |
    +-----+------------+----------------+-----------+
    |   1 | Washington | George         | 1789-1797 |
    |   2 | Adams      | John           | 1797-1801 |
    ...
    |   7 | Jackson    | Andrew         | 1829-1837 |
    ...
    |  17 | Johnson    | Andrew         | 1865-1869 |
    ...
    |  36 | Johnson    | Lyndon B.      | 1963-1969 |
    ...
    

("Andrew Johnson" đã được chọn làm câu trả lời vì sự lặp lại)

INDEX nào sẽ là tối ưu nhất cho câu hỏi này? Cụ thể hơn, điều gì là tốt nhất cho:

    
        SELECT  term
            FROM  Presidents
            WHERE  last_name = 'Johnson'
              AND  first_name = 'Andrew';
    

Một vài chỉ mục để thử...
Some INDEXes to try...

* Không có INDEX nào
* INDEX(first_name), INDEX(last_name) (2 INDEX song song) 
* "Index Merge Intersect"
* INDEX(last_name, first_name) (một  index "tổng hợp") 
* INDEX(last_name, first_name, term) (một index "bao phủ") 
* Các biến thể

## No indexes

Tôi chỉ giả sử một chút ở đây. Tôi có một khóa chính trong `seq`, nhưng lại không có lợi ích gì trong những truy vấn mà chúng ta đang nghiên cứu.

        
    mysql>  SHOW CREATE TABLE Presidents G
    CREATE TABLE `presidents` (
      `seq` tinyint(3) unsigned NOT NULL AUTO_INCREMENT,
      `last_name` varchar(30) NOT NULL,
      `first_name` varchar(30) NOT NULL,
      `term` varchar(9) NOT NULL,
      PRIMARY KEY (`seq`)
    ) ENGINE=InnoDB AUTO_INCREMENT=45 DEFAULT CHARSET=utf8
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew';
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    | id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra       |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    |  1 | SIMPLE      | Presidents | ALL  | NULL          | NULL | NULL    | NULL |   44 | Using where |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    
    # Or, using the other form of display:  EXPLAIN ... G
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ALL        <-- Implies table scan
    possible_keys: NULL
              key: NULL       <-- Implies that no index is useful, hence table scan
          key_len: NULL
              ref: NULL
             rows: 44         <-- That's about how many rows in the table, so table scan
            Extra: Using where
    

## Triển khai chi tiết

Đầu tiên, hãy mô tả InnoDB dự trữ và sử dụng index như thế nào.

* Dữ liệu và khóa chính được "nhóm" lại vs nhau trong BTree
* Tra cứu BTree rất nhanh và hiệu quả. Đối với một bản có 1 triệu bản ghi có thể được chia thành 3 mức độ của BTree, và 2 cấp cao nhất có thể được cache.
* Mỗi index thứ 2 là một BTree khác, với khóa chính ở lá.
* Tìm nạp các mục "liên tiếp" (theo index) từ một BTree rất hiệu quả bở vì chúng được lưu liên tiếp.
* Để đơn giản nó đi, chúng ta có thể đếm từng tra cứu BTree dưới dạng 1 đơn vị công việc, và loại bỏ các lượt quét chỉ mục liên tiếp. Điều này sấp xỉ số lần truy cập vào ổ đĩa cho một bảng lớn trong một hệ thống lớn.

Đối với MyISAM, khóa chính không được lưu cùng dữ liệu, vì vậy hãy nghĩ rằng nó là một khóa thứ cập (Quá đơn giản).

## INDEX(first_name), INDEX(last_name)

Đối với người mới, một khi anh ấy học cách đánh dấu index, quyết định index nhiều cột trong một thời điểm. Nhưng...

MySQL hiếm khi sử dụng nhiều hơn 1 index trong một thời điểm và 1 truy vấn.Vì vậy, nó sẽ phân tích những index có thể.

* first_name -- có 2 dòng có thể (1 là tra cứu BTree, sau đó quét liên tiếp)
* last_name -- có 2 dòng có thể. Giả sử nó lấy last_name. Đây là những bước tiếp theo trong công việc SELECT:  
1\. Sử dụng INDEX(last_name), tìm 2 index với last_name = 'Johnson'
2\. Lấy khóa chính (ngầm được thêm vào mỗi chỉ số phụ trong InnoDB); lấy (17,36)
3\. Tiếp cận dữ liệu bằng cách sử dụng seq = (17, 36) để lấy các hàng cho Andrew Johnson và Lyndon B. Johnson. 
4\. Sử dụng phần còn lại của mệnh đề Where để lọc loại bỏ lấy các hàng mong muốn.
5\. Cung cấp câu trả lời (1865-1869). 
    
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew'  G
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: last_name, first_name
              key: last_name
          key_len: 92                 <-- VARCHAR(30) utf8 may need 2+3*30 bytes
              ref: const
             rows: 2                  <-- Two 'Johnson's
            Extra: Using where
    

## "Hợp nhất các chỉ mục"

Tốt thôi, nếu bạn thực sự thông minh và quyết định MySQL có đủ thông minh để sử dụng cùng 1 lúc 2 tên chỉ mục để lấy câu trả lời. Cái này gọi là "giao điểm".
 
 1\. Sử dụng INDEX(last_name), tìm 2 chỉ mục có mục với last_name = 'Johnson'; lấy (7, 17) 
 
 2\. Sử dụng INDEX(first_name), tìm 2 chỉ mục có mục với first_name = 'Andrew'; lấy (17, 36) 
 
 3\. "And" (Hợp nhất) hai danh sách lại với nhau là (7,17) và (17,36) = (17) 
 
 4\.Tiếp cận dữ liệu bằng cách sử dụng seq = (17) để lấy các row cho Andrew Johnson. 
 
 5\. Gửi lại câu trả lời (1865-1869).
    
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: index_merge
    possible_keys: first_name,last_name
              key: first_name,last_name
          key_len: 92,92
              ref: NULL
             rows: 1
            Extra: Using intersect(first_name,last_name); Using where
    
EXPLAIN không cung cấp thông tin chi tiết về số lượng hàng được thu thập từ mỗi chỉ mục, v.v.

## INDEX(last_name, first_name)

Cái này có thể gọi là chỉ mục "hợp chất" hoặc "hỗn hợp" từ khi nó có nhiều hơn một cột.
 
 1\. Drill down the BTree for the index to get to exactly the index row for Johnson+Andrew; get seq = (17). 
 
 2\. Reach into the data using seq = (17) to get the row for Andrew Johnson. 
 
 3\. Gửi lại các câu trả lời (1865-1869). Thế này tốt hơn. Trong thực tế thì đây có thể là cách "tốt nhất".
    
    
        ALTER TABLE Presidents
            (drop old indexes and...)
            ADD INDEX compound(last_name, first_name);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: compound
              key: compound
          key_len: 184             <-- The length of both fields
              ref: const,const     <-- The WHERE clause gave constants for both
             rows: 1               <-- Goodie!  It homed in on the one row.
            Extra: Using where
    

## "Bao đóng": INDEX(last_name, first_name, term)

Ngạc nhiên chưa! Chúng ta hoàn toàn có thể làm nó tốt hơn 1 chút. Một chỉ mục "bao đóng" là một trong tất cả các trường của mệnh đề SELECT có thể tìm thấy được. Nó được thêm vào mà không cần tiếp cận các dữ liệu để hoàn thành nhiệm vụ.
 
 1\. Đào sâu xuống dưới của BTree để xác định các dòng chỉ mục 1 cách chính xác cho Johnson+Andrew; lấy seq = (17). 
 
 2\. Gửi lại câu trả lời (1865-1869). Cái "dữ liệu" BTree không được đụng vào; đây là một trong những cải tiến so với "hợp nhất".
    
    
        ... ADD INDEX covering(last_name, first_name, term);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: covering
              key: covering
          key_len: 184
              ref: const,const
             rows: 1
            Extra: Using where; Using index   <-- Note
    

Mọi thứ đề tương tự như khi sử dụng "hợp nhất", ngoại trừ việc bổ sung thêm "sử dụng các chỉ mục".

## Các biến thể

* Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong mệnh đề WHERE?Câu trả lời là: Thứ tự của các phần được AND với nhau không quan trọng. 
* Điều gì sẽ xảy ra nếu bạn xáo trộn các trường trong mệnh đề Index? Câu trả lời là: Nó có thể tạo ra những thay đổi đáng kể. Hơn cái một chút nhiều. 
* Điều gì sẽ xảy ra nếu bổ sung thêm các trường ở cuối? Câu trả lời là: Tác hại khá nhỏ; cũng có thể có rất nhiều (ví dụ như, 'covering'). 
* Dư thừa? Với điều này, nếu bạn có cả 2 cái sau: INDEX(a), INDEX(a,b)? Câu trả lời là: Việc dư thừa chi phí trên mệnh đề INSERTs; nó rất hiếm khi hữu ích đối với SELECTs. 
* Tiền tố? Với vấn đề này, INDEX(last_name(5). first_name(5)) Câu trả lời là: Không cần bận tâm; nó hiếm khi có ích, và thường gây hại hơn. (Chi tiết sẽ được đề cập trong 1 chủ đề khác.) 

## Thêm các ví dụ:
    
    
        INDEX(last, first)
        ... WHERE last = '...' -- good (even though `first` is unused)
        ... WHERE first = '...' -- index is useless
    
        INDEX(first, last), INDEX(last, first)
        ... WHERE first = '...' -- 1st index is used
        ... WHERE last = '...' -- 2nd index is used
        ... WHERE first = '...' AND last = '...' -- either could be used equally well
    
        INDEX(last, first)
        Both of these are handled by that one INDEX:
        ... WHERE last = '...'
        ... WHERE last = '...' AND first = '...'
    
        INDEX(last), INDEX(last, first)
        In light of the above example, don't bother including INDEX(last).
    

 

