# WiFi-Speed
C++ implements network speed measurement.

There are many comments in the code, which are highly readable, but they are in Chinese (English was not particularly good when I was developing). If you are Chinese or understand Chinese, you can look at the comments.
# Note
Note that Windows is not supported because Linux command line commands are used in the code. You can also use macOS or other UNIX like operating systems, but Linux is recommended (who knows what happens when programs run on those miscellaneous operating systems).

# Code
```cpp
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
 
//为了兼容C++
#undef bool
#undef true
#undef false
#define bool int
#define true 1
#define false 0
 
#define NET_INFO_PATH "/proc/net/dev"
 
#define NET_DIFF_TIME 1  //时间差阈值(单位：秒)
 
//全局参数：以供随时调用函数，随时返回值（存在延时1s以内）
static time_t net_previous_timeStamp;//记录上一次系统时间戳   时间单位:秒
static time_t net_current_timeStamp;//记录当前系统时间戳 时间单位:秒
static double net_dif_time;//记录时间差 时间单位:秒
 
static char net_file[64] = {NET_INFO_PATH};//要读取的目标文件名
static int net_off;//记录文件游标当前偏移位置
static int line_num;//记录行数
static FILE *net_stream; //访问虚拟文件系统的文件流
static char net_buffer[256];//行缓冲区 
static char *net_line_return;//记录读取每行的时候的返回结果（NULL或者返回 net_buffer）
static char net_tmp_itemName[32];//临时存放文件中的每行的项目名称
 
static int net_itemReceive;//存放每一个网卡的接受到的字节总数（单位：Byte）
static int net_itemTransmit;//存放每一个网卡的已发送的字节总数（单位：Byte）
 
static int net_current_receive_total;//存放【当前】收到的网络包字节总数（单位：Byte）
static int net_previous_receive_total;//存放【上一次】收到的网络包字节总数（单位：Byte）
static int net_receive_speed;//平均接受网络字节速度
 
static int net_current_transmit_total; //存放【当前】已发送的网络包字节总数（单位：Byte）
static int net_previous_transmit_total;//存放【上一次】已发送的网络包字节总数（单位：Byte）
static int net_transmit_speed;//平均发送网络字节速度
 
static float net_averageLoad_speed; //网络负载情况（单位：Bytes/s）
//(net_current_receive_total - net_previous_receive_total) + (net_current_transmit_total - net_previous_transmit_total) / 2
static float net_result; //函数返回值
 
//声明所有函数体
//注意：所有的log相关的执行程序，仅仅是打印日志，方便读者查看监测细节，如需实用时，除去log/lineLog相关代码即可，不会影响程序正常执行
void net_update(bool log);//初始化【必选】
void net_init(bool log, bool lineLog);//更新网络各参数信息【必选】
float net_usage(char * TYPE, bool log, bool lineLog);//通过TYPE值，获取网络信息的指定参数【必选】
void net_close();//关闭网络监测【必选】
 
 
//更新网络各信息
void net_update(bool log){// log:是否按行打印日志
    net_previous_receive_total = net_current_receive_total;//初始化上一次接收总数据
    net_previous_transmit_total = net_current_transmit_total;//初始化上一次发送总数据
    net_current_receive_total = 0;  //对当前网络接收数据总数清零，重新计算
    net_current_transmit_total = 0;//对当前网络发送数据总数清零，重新计算
     
    //获取网络的初始化数据
    /*step1: 获取并重置游标位置为文件开头处（注意：此处是大坑  原因：每次更新计算机的网络的信息时，必须重新读入，或者重置游标的默认位置）*/
    net_off = fseek(net_stream, 0, SEEK_SET);//获取并重置游标位置为文件流开头处，SEEK_CUR当前读写位置后增加net_off个位移量,重置游标位置比重新打开并获取文件流的资源要低的道理，应该都懂吧 0.0
 
    line_num = 1;//重置行数
    net_line_return = fgets (net_buffer, sizeof(net_buffer), net_stream);//读取第一行
    //printf("[net_update] line %d: %s\n", line_num, net_line_return);
    line_num++;
    net_line_return = fgets (net_buffer, sizeof(net_buffer), net_stream);//读取第二行
    //printf("[net_update] line %d: %s\n", line_num, net_line_return);
     
    net_itemReceive = 0;
    net_itemTransmit = 0;
    int flag = 1;
    while(flag == 1){
        line_num++;
        net_line_return = fgets (net_buffer, sizeof(net_buffer), net_stream);
        net_itemReceive = 0;
        net_itemTransmit = 0;
        if(net_line_return != NULL){
            sscanf( net_buffer,
                "%s%d%d%d%d%d%d%d%d%d",
                net_tmp_itemName,
                &net_itemReceive,
                &net_itemTransmit,
                &net_itemTransmit,
                &net_itemTransmit,
                &net_itemTransmit,
                &net_itemTransmit,
                &net_itemTransmit,
                &net_itemTransmit,
                &net_itemTransmit);
            net_current_receive_total += net_itemReceive;//初始化接收字节总数
            net_current_transmit_total += net_itemTransmit;//初始化发送字节总数
        } else {
            flag = -1;     
        }
 
        if(log == true){
            if(flag != -1)//如果当前行不是末尾行时，不添加换行符
                printf("[net_update] line %d: %s", line_num, net_line_return);
            else //反之，添加换行符
                printf("[net_update] line %d: %s\n", line_num, net_line_return);
            printf("[net_update] line %d:net_itemReceive: %d\n", line_num, net_itemReceive);
            printf("[net_update] line %d:net_itemTransmit: %d\n", line_num, net_itemTransmit);
            printf("[net_update] line %d:net_current_receive_total: %d\n", line_num, net_current_receive_total);
            printf("[net_update] line %d:net_current_transmit_total: %d\n\n", line_num, net_current_transmit_total);
        }
    }  
}
 
//初始化
void net_init(bool log, bool lineLog){//log:打印更新的【网络各参数信息】,以便检测细节； lineLog：按行打印网络文件信息细节
    //读取数据
    net_line_return = "INIT"; //设置初始值，只要不为空字符串即可
    /*step1: 打开文件 （注意：此处是大坑  原因：每次更新计算机的网络的信息时，必须重新读入，或者重置游标的默认位置）*/
    net_stream = fopen (net_file, "r"); //以R读的方式打开文件再赋给指针fd
    net_off = fseek(net_stream, 0, SEEK_SET);//获取并重置游标位置为文件流开头处，SEEK_CUR当前读写位置后增加net_off个位移量
     
    net_update(lineLog);//更新计算机的网络的信息  
     
    net_previous_receive_total = net_current_receive_total;//初始化上一次接收总数据
    net_previous_transmit_total = net_current_transmit_total;//初始化上一次发送总数据
 
    net_receive_speed = 0;//初始化接收网速为：0
    net_transmit_speed = 0;//初始化发送网速为：0
    net_averageLoad_speed = 0.0; //初始化网络负载速度为：0
    net_previous_timeStamp = net_current_timeStamp = time(NULL);//开始获取初始化的时间戳
    net_dif_time = 0;//初始化时间差
     
    if(log == true)
    {
        printf("\n[INIT NET] net_current_timeStamp: %f\n", (double)net_current_timeStamp);
        printf("\n[INIT NET] Receive Total: %d bytes.\n", net_current_receive_total);
        printf("\n[INIT NET] Receive Speed: %d bytes/s.\n", net_receive_speed);
        printf("\n[INIT NET] Transmit Total: %d bytes.\n", net_current_transmit_total);
        printf("\n[INIT NET] Transmit Speed: %d bytes/s.\n", net_transmit_speed);
        printf("\n[INIT NET] Average Load Speed: %f bytes/s.\n", net_averageLoad_speed);
        printf("*********************************************************************************************************\n"); 
    }
}
 
//参数：TYPE:["receiveTotal/receiveSpeed/transmitTotal/transmitSpeed/averageLoadSpeed"]
float net_usage(char * TYPE, bool log, bool lineLog)
{//log:打印更新的【网络各参数信息】,以便检测细节； lineLog：按行打印网络文件信息细节
    net_current_timeStamp = time(NULL);//time() 单位：秒
    printf("[net_usage] net_current_timeStamp: %f.\n", (double)net_current_timeStamp);//time_t[long int]必须转型,否则无法正常(输出显示:0.000)
    net_dif_time = (double)(net_current_timeStamp - net_previous_timeStamp);//计算时间差（单位：秒）
    if( (net_dif_time) >= NET_DIFF_TIME ){//只有满足达到时间戳以后，才更新接收与发送的网络字节数据信息   
        net_update(lineLog);//是否按行打印更新的网络参数信息
        net_receive_speed = (net_current_receive_total - net_previous_receive_total) / NET_DIFF_TIME;//更新接收网速(单位：字节/秒)
        net_transmit_speed = (net_current_transmit_total - net_previous_transmit_total) / NET_DIFF_TIME;//更新发送网速(单位：字节/秒)
        net_averageLoad_speed = (net_receive_speed + net_transmit_speed) / 2;   //更新平均网络负载情况
        //更新时间戳
        net_previous_timeStamp = net_current_timeStamp;
    }
 
    if(log == true)
    {
        printf("\n[NET] LAST Receive Total: %d bytes [%f KB].\n", net_previous_receive_total, (double)(net_previous_receive_total / 1024));
        printf("\n[NET] NOW  Receive Total: %d bytes [%f KB].\n", net_current_receive_total, (double)(net_current_receive_total / 1024));
        printf("\n[NET] NOW Receive Speed: %d bytes/s [%f KB/s].\n", net_receive_speed, (double)(net_receive_speed / 1024));   
 
        printf("\n[NET] LAST Transmit Total: %d bytes [%f KB].\n", net_previous_transmit_total, (double)(net_previous_transmit_total / 1024));
        printf("\n[NET] NOW  Transmit Total: %d bytes [%f KB].\n", net_current_transmit_total, (double)(net_current_transmit_total / 1024));
        printf("\n[NET] NOW Transmit Speed: %d bytes/s [%f KB/s].\n", net_transmit_speed, (double)(net_transmit_speed / 1024));
        printf("\n[NET] Average Load Speed: %f bytes/s [%f KB/s].\n", net_averageLoad_speed, (double)(net_averageLoad_speed / 1024));  
        printf("*********************************************************************************************************\n");
    }
     
    if(TYPE == "receiveTotal"){
        net_result = (float)net_current_receive_total;
    } else if(TYPE == "receiveSpeed"){
        net_result = (float)net_receive_speed; 
    } else if(TYPE == "transmitTotal"){
        net_result = (float)net_current_transmit_total;
    } else if(TYPE == "transmitSpeed"){
        net_result = (float)net_transmit_speed;
    } else if(TYPE == "averageLoadSpeed"){
        net_result = net_averageLoad_speed;
    } else {// TYPE 不存在时，抛出异常值 -1
        net_result = -1;   
    }
 
    return net_result;
}
 
//关闭网络监控
void net_close(){
    fclose(net_stream);     //关闭文件net_stream
}

void calculate(bool initLog, bool usageLog, bool lineLog){
    printf("\n[MAIN] receiveTotal:%f\n", net_usage("receiveTotal", usageLog, lineLog));//log:打印日志在屏幕上：以便检测细节
    printf("*********************************************************************************************************\n"); 
    sleep(3);  
    printf("\n[MAIN] receiveTotal:%f\n", net_usage("receiveTotal", usageLog, lineLog));//log:打印日志在屏幕上：以便检测细节
    printf("*********************************************************************************************************\n");
}
 
void execute(bool initLog, bool usageLog, bool lineLog){//
    system("cat /proc/net/dev");
    printf("*********************************************************************************************************\n");
    net_init(initLog, lineLog);
    calculate(initLog, usageLog, lineLog);
    net_close();
}
 
int main()
{
    execute(true, true, true);//测试用例1
    printf("########################################################################################################\n");
    execute(false, true, false);//测试用例2
    return 0;
}
```
