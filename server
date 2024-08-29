#include <iostream>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <cstring>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <pthread.h>
#include <map>
#include <mariadb/conncpp.hpp>

#define BUF_SIZE 100
#define MAX_CLNT 256
using namespace std;

char wcome[20] = "[DEFAULT]";

void *handle_clnt(void *arg);
void rand_msg(const char *msg, int len);
void error_handling(const char *msg);
void addUser(std::string NAME, std::string NICNAME, std::string ID, std::string PW);
void Userlogin(string user_id, string user_pw, int clnt_sock);

std::vector<int> buddy_socks(MAX_CLNT);
int buddy_sock_cnt = 0;
int clnt_cnt = 0;
int clnt_socks[MAX_CLNT];
pthread_mutex_t mutx;
std::map<int, std::string> sock_to_nick; // 소켓 번호와 닉네임 일치 맵

class DB
{
protected:
    sql::Connection *conn;

public:
    sql::PreparedStatement *prepareStatement(const string &query) // ? 쿼리문 삽입
    {
        sql::PreparedStatement *stmt(conn->prepareStatement(query));
        return stmt;
    }

    sql::Statement *createStatement() // 쿼리문 즉시 실행
    {
        sql::Statement *stm(conn->createStatement());
        return stm;
    }

    void connect()
    {
        try
        {
            sql::Driver *driver = sql::mariadb::get_driver_instance();
            sql::SQLString url("jdbc:mariadb://10.10.21.111:3306/CHAT");
            sql::Properties properties({{"user", "CHAT"}, {"password", "1"}});
            conn = driver->connect(url, properties);
        }
        catch (sql::SQLException &e)
        {
            std::cerr << "Error Connecting to MariaDB Platform: " << e.what() << std::endl;
        }
    }

    void addUser(std::string NAME, std::string NICNAME, std::string ID, std::string PW)
    {
        try
        {
            std::unique_ptr<sql::PreparedStatement> stmnt(conn->prepareStatement("insert into USER values (default,?, ?, ?, ?, 0, 0, NULL)"));
            stmnt->setString(1, NAME);
            stmnt->setString(2, NICNAME);
            stmnt->setString(3, ID);
            stmnt->setString(4, PW);
            stmnt->executeQuery();
        }
        catch (sql::SQLException &e)
        {
            std::cerr << "Error inserting new task: " << e.what() << std::endl;
        }
    }

    void Userlogin(string user_id, int clnt_sock)
    {
        try
        {
            cout << user_id << endl;

            string clnt_sock_no = to_string(clnt_sock);
            cout << "소켓 번호" << clnt_sock << endl;

            std::unique_ptr<sql::PreparedStatement> stmnt9(conn->prepareStatement("UPDATE USER SET STATUS = '0', SOKET_NUM = NULL  WHERE STATUS = '1' AND USER_ID = '" + user_id + "';"));
            stmnt9->executeQuery(); // 유저 접속시 접속중으로 바꿈

            std::unique_ptr<sql::PreparedStatement> stmnt(conn->prepareStatement("UPDATE USER SET STATUS = '1', SOKET_NUM = '" + clnt_sock_no + "' WHERE STATUS = '0' AND USER_ID = '" + user_id + "' AND SOKET_NUM IS NULL"));
            stmnt->executeQuery(); // 유저 접속시 접속중으로 바꿈
            return;
        }
        catch (sql::SQLException &e)
        {
            std::cerr << "Error inserting new task: " << e.what() << std::endl;
        }
    }

    bool attemptLogin(string user_id, string user_pw, int clnt_sock)
    {
        try
        {
            string userno2, set2;
            std::unique_ptr<sql::Statement> stmnt(conn->createStatement());
            sql::ResultSet *res = stmnt->executeQuery("SELECT * FROM USER");
            int i = 4;
            while (res->next())
            {
                if (res->getString(4) == user_id && res->getString(5) == user_pw)
                {
                    userno2 = res->getString(1);
                    set2 = res->getString(3);

                    string message = "로그인 성공";
                    write(clnt_sock, message.c_str(), message.size());
                    // sleep(1);
                    usleep(100);
                    write(clnt_sock, userno2.c_str(), userno2.size());
                    std::cout << userno2 << " 고유번호 전송." << endl;
                    // sleep(1);
                    usleep(100);
                    write(clnt_sock, set2.c_str(), set2.size());
                    std::cout << set2 << " 님이 접속하셨습니다." << endl;

                    return true; // 1
                }
            }
        }
        catch (sql::SQLException &e)
        {
            std::cerr << "Error during login attempt: " << e.what() << std::endl;
        }

        return false; // 0
    }

    bool Reset()
    {
        try
        {
            std::unique_ptr<sql::Statement> stmnt(conn->createStatement());
            sql::ResultSet *res = stmnt->executeQuery("SELECT * FROM USER");
            while (res->next())
            {
                if (res->getString(6) == "1")
                {
                    return true;
                }
            }
        }
        catch (sql::SQLException &e)
        {
            std::cerr << "Error during login attempt: " << e.what() << std::endl;
        }

        return false;
    }

    ~DB() { delete conn; }
};

class USER
{
private:
    DB &USERDB;
    string user_id, user_pw, user_NO;

public:
    USER(DB &db) : USERDB(db) {}

    void longin(int clnt_sock)
    {
        int str_len = 0;
        char msg[BUF_SIZE];
        try
        {
            while ((str_len = read(clnt_sock, msg, sizeof(msg))) != 0)
            {
                if (!strcmp(msg, "1\n"))
                {
                    int count = 0;
                    while (1)
                    {
                        ++count;
                        memset(msg, 0, BUF_SIZE);
                        read(clnt_sock, msg, sizeof(msg)); // 아이디 READ
                        user_id = std::string(msg);
                        memset(msg, 0, BUF_SIZE);
                        read(clnt_sock, msg, sizeof(msg)); // 패스워드 READ
                        user_pw = std::string(msg);
                        if (count >= 6)
                        {
                            memset(msg, 0, BUF_SIZE);
                            read(clnt_sock, msg, sizeof(msg));
                            user_NO = std::string(msg);
                        }
                        memset(msg, 0, BUF_SIZE);
                        int logbool = USERDB.attemptLogin(user_id, user_pw, clnt_sock);
                        if (logbool == true)
                        {
                            USERDB.Userlogin(user_id, clnt_sock);
                            while (1)
                            {
                                string mge = "1번 : 친구관리    2번 : 1:1채팅    3번 : 랜덤채팅    4번 : 그룹채팅   5번 : 친구 요청 창";
                                write(clnt_sock, mge.c_str(), mge.size()); // 선택지 보여줌

                                memset(msg, 0, BUF_SIZE);
                                read(clnt_sock, msg, sizeof(msg)); // 고른 번호 입력받기

                                if (!strcmp(msg, "1\n"))
                                {
                                     AddBuddy(clnt_sock);
                                }
                                else if (!strcmp(msg, "2\n"))
                                {
                                    mtm_chat(clnt_sock);
                                }
                                else if (!strcmp(msg, "3\n"))
                                {
                                    Randchat(clnt_sock);
                                }
                                else if (!strcmp(msg, "4\n"))
                                {
                                    group_chat(clnt_sock);
                                }
                                else if(!strcmp(msg,"5\n"))
                                {
                                    Buddyupdate(clnt_sock);
                                }
                            }
                            break;
                        }
                        else if (count < 5)
                        {
                            cout << "로그인 실패" << count << endl;
                            string message = "로그인 실패. 재시도하세요.";
                            write(clnt_sock, message.c_str(), message.size());
                        }
                        else
                        {
                            cout << "로그인 실패, 고유번호" << endl;
                            string message = "5회 실패. 고유번호를 입력하세요.";
                            write(clnt_sock, message.c_str(), message.size());
                        }
                    }
                }

                if (!strcmp(msg, "2\n"))
                {
                    cout << "2번 선택" << endl;
                    string id, pw, NAME, NICNAME;
                    for (int i = 0; i < 4; i++)
                    {
                        str_len = read(clnt_sock, msg, sizeof(msg));
                        msg[str_len] = '\0';

                        if (i == 0)
                        {
                            id = string(msg);
                        }
                        else if (i == 1)
                        {
                            pw = string(msg);
                        }
                        else if (i == 2)
                        {
                            NAME = string(msg);
                        }
                        else if (i == 3)
                        {
                            NICNAME = string(msg);
                        }
                    }

                    std::unique_ptr<sql::Statement> stmnt(USERDB.createStatement());
                    sql::ResultSet *res = stmnt->executeQuery("select * from USER");

                    while (res->next())
                    {
                        if (id == res->getString(4))
                        {
                            string check = "1";
                            write(clnt_sock, check.c_str(), check.size());

                            for (int i = 0; i < 4; i++)
                            {
                                str_len = read(clnt_sock, msg, sizeof(msg));
                                msg[str_len] = '\0';

                                if (i == 0)
                                {
                                    id = string(msg);
                                }
                                else if (i == 1)
                                {
                                    pw = string(msg);
                                }
                                else if (i == 2)
                                {
                                    NAME = string(msg);
                                }
                                else if (i == 3)
                                {
                                    NICNAME = string(msg);
                                }
                            }
                            break;
                        }
                    }

                    USERDB.addUser(NAME, NICNAME, id, pw);
                }
            }
        }
        catch (sql::SQLException &e)
        {
            std::cerr << "Error Connecting to MariaDB Platform: " << e.what() << std::endl;
        }
    }

    void Randchat(int clnt_sock)
    {
        cout << "랜챗 진입2" << endl;
        int chat_socks[2];
        char msg[BUF_SIZE];
        string message;
        try
        {
            string str1 = to_string(clnt_sock);
            std::unique_ptr<sql::Statement> stmnt(USERDB.createStatement());
            sql::ResultSet *res = stmnt->executeQuery("SELECT USER_NO, USER_NAME, USER_ID, SOKET_NUM FROM USER WHERE RAND_CHAT = 1 AND SOKET_NUM != '" + str1 + "' ORDER BY RAND();");
            cout << "랜덤 쿼리" << endl;

            pthread_mutex_lock(&mutx);
            chat_socks[0] = clnt_sock;
            pthread_mutex_unlock(&mutx);

            while (res->next())
            {
                cout << "랜챗 진입1" << endl;
                string rensponse = "1번 : 랜덤 상대 연결   2번 : 대화 종료";
                write(clnt_sock, rensponse.c_str(), rensponse.size());
                memset(msg, 0, BUF_SIZE);

                read(clnt_sock, msg, sizeof(msg));

                int connect_sock = res->getInt(4);
                pthread_mutex_lock(&mutx);
                chat_socks[1] = connect_sock;
                pthread_mutex_unlock(&mutx);

                if (!strcmp(msg, "1\n"))
                {
                    int str_len = 0;
                    message = "랜덤 채팅 상대를 찾았습니다. (다음 상대 찾기 q or Q)\n";
                    write(clnt_sock, message.c_str(), message.size());
                    write(connect_sock, message.c_str(), message.size());
                    while (1)
                    {
                        memset(msg, 0, BUF_SIZE);
                        int len = read(clnt_sock, msg, BUF_SIZE);
                        string ranchat_response(msg, len);
                        if (ranchat_response == "q\n" || ranchat_response == "Q\n")
                        {
                            string message = "상대방이 나갔습니다.\n";
                            write(connect_sock, message.c_str(), message.size());
                            break;
                        }
                        write(connect_sock, msg, len);

                        int len2 = read(connect_sock, msg, BUF_SIZE);
                        string ranchat_response2(msg, len2);
                        if (ranchat_response2 == "q" || ranchat_response2 == "Q")
                        {
                            string message = "상대방이 나갔습니다.\n";
                            write(clnt_sock, message.c_str(), message.size());
                            break;
                        }
                        write(clnt_sock, msg, len2);
                    }
                }
                else if (!strcmp(msg, "2\n"))
                {
                    message = "대화를 종료합니다.\n";
                    write(clnt_sock, message.c_str(), message.size());
                    break;
                }
            }
        }
        catch (sql::SQLException &e)
        {
            std::cerr << "Error during random chat: " << e.what() << std::endl;
        }
    }

    void group_chat(int clnt_sock)
    {
        std::vector<std::string> buddy(MAX_CLNT);
        std::string user_no;
        char msg[120];
        char group_msg[120];
        int buddy_cnt = 0;
        int buddy_sock;
        int buddy_choice = 0;
        std::string buddy_name;
        std::string buddy_nickname;
        std::string user_nickname;
        std::string left = " (";
        std::string right = ") ";
        std::string colon = " : ";
        int str_len = 0;

        try
        {
            std::unique_ptr<sql::Statement> stmnt(USERDB.createStatement());
            sql::ResultSet *res(stmnt->executeQuery("SELECT * FROM USER"));

            std::string group_chat_msg = "그룹채팅입니다.\n";
            write(clnt_sock, group_chat_msg.c_str(), group_chat_msg.size());

            while (res->next())
            {
                if (res->getString(4) == user_id)
                {
                    user_no = res->getString(1);
                    user_nickname = res->getString(3);
                }
            }

            std::unique_ptr<sql::Statement> stmnt2(USERDB.createStatement());
            sql::ResultSet *res1(stmnt2->executeQuery("SELECT * FROM BUDDY WHERE USER_NUM =" + user_no));

            group_chat_msg = "대화가능한 친구 목록 입니다\n";
            write(clnt_sock, group_chat_msg.c_str(), group_chat_msg.size());
            while (res1->next())
            {
                if (res1->getBoolean(7) && res1->getInt(8) == 1)
                {
                    buddy_name = res1->getString(5);
                    buddy_nickname = res1->getString(6);
                    buddy_cnt++;
                    std::string ok_buddy = std::to_string(buddy_cnt) + colon + buddy_name + left + buddy_nickname + right + "\n";
                    write(clnt_sock, ok_buddy.c_str(), ok_buddy.size());

                    buddy[buddy_cnt] = buddy_nickname;
                }
            }

            while (1)
            {
                group_chat_msg = "초대할 친구를 골라주세요. 대화를 시작하려면 t\n";
                write(clnt_sock, group_chat_msg.c_str(), group_chat_msg.size());
                memset(group_msg, 0, BUF_SIZE);
                read(clnt_sock, group_msg, sizeof(group_msg));
                if (!strcmp(group_msg, "t\n"))
                    break;
                buddy_choice = std::stoi(group_msg);
            }

            for (int i = 1; i <= buddy_choice; ++i)
            {
                buddy_nickname = buddy[i];
                std::unique_ptr<sql::Statement> stmnt3(USERDB.createStatement());
                sql::ResultSet *res3(stmnt3->executeQuery("SELECT SOKET_NUM FROM USER WHERE USER_NICKNAME = '" + buddy_nickname + "'"));
                if (res3->next())
                {
                    buddy_sock = res3->getInt(1);
                    buddy_socks[buddy_sock_cnt++] = buddy_sock;
                }
            }

            group_chat_msg = "그룹채팅을 시작합니다.\n";
            write(clnt_sock, group_chat_msg.c_str(), group_chat_msg.size());

            char username[200];
            while ((str_len = read(clnt_sock, msg, sizeof(msg) - 1)) != 0)
            {
                msg[str_len] = '\0';
                sprintf(username, "[%s] %s", user_nickname.c_str(), msg);
                string group_response(msg, str_len);
                if (group_response == "q\n" || group_response == "Q\n")
                {
                    string message = "상대방이 나갔습니다.\n";
                    send_msg(message.c_str(), message.size());
                    break;
                }
                send_msg(username, strlen(username));
            }

            pthread_mutex_lock(&mutx);
            for (int i = 0; i < clnt_cnt; i++)
            {
                if (clnt_sock == clnt_socks[i])
                {
                    while (i++ < clnt_cnt - 1)
                        clnt_socks[i] = clnt_socks[i + 1];
                    break;
                }
            }
            clnt_cnt--;
            pthread_mutex_unlock(&mutx);
        }

        catch (sql::SQLException &e)
        {
            std::cerr << "Error during random chat: " << e.what() << std::endl;
        }
    }

    void send_msg(const char *msg, int len)
    {
        int i;
        pthread_mutex_lock(&mutx);
        for (i = 0; i < clnt_cnt; i++)
            write(clnt_socks[i], msg, len);
        pthread_mutex_unlock(&mutx);
    }
    void mtm_chat(int clnt_sock)
    {
        std::vector<std::string> buddy(MAX_CLNT);
        std::string user_no;
        char msg[120];
        char mtm_msg[120];
        int buddy_cnt = 0;
        int buddy_sock = 0;
        int buddy_choice = 0;
        std::string buddy_name;
        std::string buddy_nickname;
        std::string user_nickname;
        std::string left = " (";
        std::string right = ") ";
        std::string colon = " : ";
        int str_len = 0;

        try
        {
            std::unique_ptr<sql::Statement> stmnt(USERDB.createStatement());
            sql::ResultSet *res(stmnt->executeQuery("SELECT * FROM USER"));

            std::string mtm_chat_msg = "1:1채팅입니다.\n";
            write(clnt_sock, mtm_chat_msg.c_str(), mtm_chat_msg.size());

            while (res->next())
            {
                if (res->getString(4) == user_id)
                {
                    user_no = res->getString(1);
                    user_nickname = res->getString(3);
                }
            }

            std::unique_ptr<sql::Statement> stmnt2(USERDB.createStatement());
            sql::ResultSet *res1(stmnt2->executeQuery("SELECT * FROM BUDDY WHERE USER_NUM =" + user_no));

            mtm_chat_msg = "대화가능한 친구 목록 입니다\n";
            write(clnt_sock, mtm_chat_msg.c_str(), mtm_chat_msg.size());
            while (res1->next())
            {
                if (res1->getBoolean(7) && res1->getInt(8) == 1)
                {
                    buddy_name = res1->getString(5);
                    buddy_nickname = res1->getString(6);
                    buddy_cnt++;
                    std::string ok_buddy = std::to_string(buddy_cnt) + colon + buddy_name + left + buddy_nickname + right + "\n";
                    write(clnt_sock, ok_buddy.c_str(), ok_buddy.size());

                    buddy[buddy_cnt] = buddy_nickname;
                }
            }

            mtm_chat_msg = "대화할 친구를 골라주세요.\n";
            write(clnt_sock, mtm_chat_msg.c_str(), mtm_chat_msg.size());
            memset(mtm_msg, 0, BUF_SIZE);
            read(clnt_sock, mtm_msg, sizeof(mtm_msg));
            buddy_choice = std::stoi(mtm_msg);

            for (int i = 1; i <= buddy_choice; ++i)
            {
                buddy_nickname = buddy[i];
                std::unique_ptr<sql::Statement> stmnt3(USERDB.createStatement());
                sql::ResultSet *res6(stmnt3->executeQuery("SELECT SOKET_NUM FROM USER WHERE USER_NICKNAME = '" + buddy_nickname + "';"));
                if (res6->next())
                {
                    buddy_sock = res6->getInt(1);
                    buddy_socks[buddy_sock_cnt++] = buddy_sock;
                }
            }
            mtm_chat_msg = "1:1채팅을 시작합니다.";
            write(clnt_sock, mtm_chat_msg.c_str(), mtm_chat_msg.size());

            char username[200];
            while ((str_len = read(clnt_sock, msg, sizeof(msg) - 1)) != 0)
            {
                msg[str_len] = '\0';
                sprintf(username, "[%s] %s", user_nickname.c_str(), msg);
                string mtm_response(msg, str_len);
                if (mtm_response == "q\n" || mtm_response == "Q\n")
                {
                    string message = "상대방이 나갔습니다.\n";
                    write(buddy_sock, message.c_str(), message.size());
                    write(clnt_sock, message.c_str(), message.size());
                    break;
                }
                write(buddy_sock, username, strlen(username));
                write(clnt_sock, username, strlen(username));
            }
        }
        catch (sql::SQLException &e)
        {
            std::cerr << "Error inserting new task: " << e.what() << std::endl;
        }
    }
     void Buddyupdate(int clnt_sock)
    {
        vector<std::string> buddy(MAX_CLNT);
        string user_no;
        char msg[120];
        char in_msg[120];
        int buddy_cnt = 0;
        int buddy_sock;
        int buddy_choice = 0;
        string buddy_name;
        string buddy_nickname;
        string user_nickname;
        string left = " (";
        string right = ") ";
        string colon = " : ";
        int str_len = 0;
        string mynum;
        int choice;
        try
        {
            std::unique_ptr<sql::Statement> stmnt(USERDB.createStatement());
            sql::ResultSet *res(stmnt->executeQuery("SELECT * FROM USER"));

            std::string in_msg = "친구 요청 창입니다.\n";
            write(clnt_sock, in_msg.c_str(), in_msg.size());
            memset(msg, 0, BUF_SIZE);

            while (res->next())
            {
                if (res->getString(4) == user_id)
                {
                    user_no = res->getString(1);
                    user_nickname = res->getString(3);
                }
            }

            std::unique_ptr<sql::Statement> stmnt2(USERDB.createStatement());
            sql::ResultSet *res1(stmnt2->executeQuery("SELECT * FROM BUDDY WHERE USER_NUM =" + user_no));

            in_msg = "요청한 친구 목록 입니다\n";
            cout << in_msg << endl;
            write(clnt_sock, in_msg.c_str(), in_msg.size());

            while (res1->next())
            {
                if (res1->getInt(8) == 3)
                {
                    buddy_name = res1->getString(5);
                    buddy_nickname = res1->getString(4);
                    buddy_cnt++;
                    std::string ok_buddy = std::to_string(buddy_cnt) + colon + buddy_name + left + buddy_nickname + right + "\n";
                    write(clnt_sock, ok_buddy.c_str(), ok_buddy.size());
                    buddy[buddy_cnt] = buddy_nickname;
                }
            }
            memset(msg, 0, BUF_SIZE);
            read(clnt_sock, msg, sizeof(msg));
            string bmsg = msg;
            cout << bmsg << endl;
            // std::unique_ptr<sql::PreparedStatement> stmnt4(USERDB.prepareStatement("UPDATE BUDDY SET BUDDY_BUF = ? WHERE BUDDY_NUM = ?;"));
            // sql::ResultSet *res4 = stmnt4->executeQuery();
            memset(msg, 0, BUF_SIZE);
            read(clnt_sock, msg, sizeof(msg));
            cout<<msg<<endl;
            choice=stoi(msg);
            while (1)
            {
                if (choice == 1)
                {
                    std::unique_ptr<sql::PreparedStatement> stmnt4(USERDB.prepareStatement("UPDATE BUDDY SET BUDDY_BUF = ? WHERE BUDDY_NUM = ?;"));
                    stmnt4->setInt(1, 1);
                    stmnt4->setString(2,bmsg);
                    sql::ResultSet *res4 = stmnt4->executeQuery();
                    break;
                }
                else if (choice == 2)
                {
                    std::unique_ptr<sql::PreparedStatement> stmnt4(USERDB.prepareStatement("UPDATE BUDDY SET BUDDY_BUF = ? WHERE BUDDY_NUM = ?;"));
                    stmnt4->setInt(1, 2);
                    stmnt4->setString(2,bmsg);
                    sql::ResultSet *res4 = stmnt4->executeQuery();
                    break;
                }
                else
                {
                    std::unique_ptr<sql::PreparedStatement> stmnt4(USERDB.prepareStatement("UPDATE BUDDY SET BUDDY_BUF = ? WHERE BUDDY_NUM = ?;"));
                    stmnt4->setInt(1, 3);
                    stmnt4->setString(2,bmsg);
                    sql::ResultSet *res4 = stmnt4->executeQuery();
                    break;
                }
            }
        }
        catch (sql::SQLException &e)
        {
            std::cerr << "Error inserting new task: " << e.what() << std::endl;
        }
    }

    void AddBuddy(int clnt_sock) // 친구추가
    {
        string yours;
        cout << "추가할 친구의 고유번호를 입력하세요 : ";
        char msg[BUF_SIZE];
        memset(msg, 0, BUF_SIZE);
        read(clnt_sock, msg, sizeof(msg));
        yours = msg;
        cout << yours << endl;
        int iu = stoi(msg);
        try
        {
            cout << iu << endl;
            std::unique_ptr<sql::PreparedStatement> stmnt(USERDB.prepareStatement("INSERT INTO BUDDY VALUES (?, ?, ?, ?, ?, ?,1,3);"));
            std::unique_ptr<sql::Statement> stmnt2(USERDB.createStatement());
            std::unique_ptr<sql::Statement> stmnt1(USERDB.createStatement());
            sql::ResultSet *res3 = stmnt1->executeQuery("SELECT * FROM USER;");
            sql::ResultSet *res5 = stmnt2->executeQuery("SELECT * FROM USER WHERE USER_NO ='" + yours + "';");

            while (res3->next())
            {
                if (user_id == res3->getString(4))
                {
                    stmnt->setString(1, res3->getString(1));
                    stmnt->setString(2, res3->getString(2));
                    stmnt->setString(3, res3->getString(3));
                    cout << iu << endl;
                }

                if (iu == res3->getInt(1))
                {
                    while (res5->next())
                    {
                        cout << "check" << endl;
                        cout << res5->getString(1) << endl;
                        stmnt->setString(4, res5->getString(1));
                        stmnt->setString(5, res5->getString(2));
                        stmnt->setString(6, res5->getString(3));
                        stmnt->executeQuery();
                    }
                }
            }
        }
        catch (sql::SQLException &e)
        {
            std::cerr << "Error inserting new task: " << e.what() << std::endl;
        }
    }
};

int main(int argc, char *argv[])
{
    int serv_sock, clnt_sock;
    struct sockaddr_in serv_adr, clnt_adr;
    socklen_t clnt_adr_sz;
    pthread_t t_id;
    if (argc != 2)
    {
        std::cout << "Usage : " << argv[0] << "<port>\n";
        exit(1);
    }

    pthread_mutex_init(&mutx, NULL);
    serv_sock = socket(PF_INET, SOCK_STREAM, 0);

    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_adr.sin_port = htons(atoi(argv[1]));

    if (bind(serv_sock, (struct sockaddr *)&serv_adr, sizeof(serv_adr)) == -1)
        error_handling("bind() error");
    if (listen(serv_sock, 5) == -1)
        error_handling("listen() error");

    while (1)
    {
        clnt_adr_sz = sizeof(clnt_adr);
        clnt_sock = accept(serv_sock, (struct sockaddr *)&clnt_adr, &clnt_adr_sz);
        std::cout << "clnt_sock : " << clnt_sock << std::endl;

        pthread_mutex_lock(&mutx);
        clnt_socks[clnt_cnt++] = clnt_sock;
        pthread_mutex_unlock(&mutx);

        pthread_create(&t_id, NULL, handle_clnt, (void *)&clnt_sock);
        pthread_detach(t_id);
        std::cout << ("Connected client IP: %s \n", inet_ntoa(clnt_adr.sin_addr));
    }
    close(serv_sock);
    return 0;
}

void *handle_clnt(void *arg)
{
    int clnt_sock = *((int *)arg);
    char msg[BUF_SIZE];
    DB db;
    USER user(db);
    db.connect();
    user.longin(clnt_sock);

    pthread_mutex_lock(&mutx);
    for (int i = 0; i < clnt_cnt; i++)
    {
        if (clnt_sock == clnt_socks[i])
        {
            while (i++ < clnt_cnt - 1)
                clnt_socks[i] = clnt_socks[i + 1];
            break;
        }
    }
    clnt_cnt--;
    pthread_mutex_unlock(&mutx);

    close(clnt_sock);
    return NULL;
}

void error_handling(const char *msg)
{
    fputs(msg, stderr);
    fputc('\n', stderr);
    exit(1);
}
