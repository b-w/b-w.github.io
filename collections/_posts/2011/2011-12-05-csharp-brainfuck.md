---
layout: post
title: "A brainfuck interpreter in C#"
categories: blog
---

You know you're spending your nights correctly if it's suddenly 1:30 in the morning and you've just spent the last several hours not only writing a brainfuck interpreter, but also programs in brainfuck for that interpreter. For the uninitiated, [brainfuck](http://en.wikipedia.org/wiki/Brainfuck) is a minimalistic toy language which is both amusing and completely unpractical (although is it Turing-complete).

The language only has eight commands, which are all single characters. The commands each manipulate what can best be thought of as a tape on a classical Turing machine. For example, the > command moves the data pointer one spot to the right. The + command increments the value at the data pointer by one. Etcetera.

Writing the interpreter is actually not the hard part. From start to finish, I completed mine in under an hour. The tricky part is writing brainfuck programs. As the name implies, brainfuck tends to be mildly difficult to understand. For example, the following brainfuck program simply prints the letter "A" to the standard output:

    ++++++
    [
        >
        +++++ +++++ +
        <
        -
    ]
    > -
    .

If your brain has ceased to function, don't worry. Help is on the way. Let's go over what this program does line by line (note: the formatting above is done for the sake of "clarity"; brainfuck simply ignores all characters, including the likes of tabs and linebreaks, that it doesn't recognize as a command):

On line 1, the value in the cell at the data pointer (currently 0) is incremented 6 times (from 0 to 6). Line 2 marks the start of brainfuck's "while loop": if the value at the data pointer is 0 it jumps the program pointer to the position after the matching \] bracket, if not it goes into the loop. Line 3 increments the data pointer by one, pointing it to cell 1\. Line 4 increments the value in cell 1 eleven times, from 0 to 11\. Line 5 decrements the data pointer by one, pointing it to cell 0 again. Line 6 decrements the value in cell 0 by one. Line 7 marks the end of the while loop: if the value at the data pointer is 0 it increments the program pointer by one, if not it jumps back to the position after the matching \[ bracket. In this case, the loop is executed 6 times. After that the program reaches line 8, which increments the data pointer by one (pointing it to cell 1) and decrements the value in this cell by one (from 66 to 65). Line 9 prints the value of the cell at the data pointer to the standard output, which effectively prints the ASCII character 65 (the "A") to the console.

Now guess what the following program does:

    +++++++++++[>++++>+++++++++>++++++++++<<<-]>++>->--<.>.+++.>++[<---->-]<.<<.>.-.>>
    ++[<+++++>-]<+.++.+++.>++[<---->-]<.---.<+++++..<.>---.>+++.--.

Oh right, you need an interpreter for that. Well, I just happen to have one.

Full code listing:

```csharp
using System;
using System.Collections.Generic;
using System.IO;

namespace BrainfuckInterpreter
{
    /// <summary>
    /// A simple but fully-featured interpreter for the Brainfuck language.
    /// </summary>
    public sealed class Program
    {
        // number of cells of the available memory
        static int TAPE_LENGTH = 32767;

        // required variables
        static string prog;
        static short[] tape;
        static int ptrTape;
        static int ptrProg;
        static Stack<int> loopStack;
        static Dictionary<int, int> jmpTable;

        static void Main(string[] args)
        {
            // the interpreter takes one argument: a path to a brainfuck source file
            if (args.Length == 0)
            {
                Console.WriteLine("USAGE: Provide path to a brainfuck source file as the first argument.");
                return;
            }

            // check if the file exists
            var sourceFile = new FileInfo(args[0]);
            if (!sourceFile.Exists)
            {
                Console.WriteLine("ERROR: File \"{0}\" not found.", sourceFile.FullName);
                return;
            }

            // alright, good to go
            Interpret(sourceFile);
        }

        static void Interpret(FileInfo sourceFile)
        {
            // try to open and read the file using a StreamReader
            try
            {
                using (StreamReader reader = sourceFile.OpenText())
                {
                    prog = reader.ReadToEnd();
                }
            }
            catch (Exception)
            {
                // something went wrong
                // most likely, this would be either a security exception or the
                // file is too big
                Console.WriteLine("ERROR: File \"{0}\" failed to read.", sourceFile.FullName);
                return;
            }

            // initialize variables
            tape = new short[TAPE_LENGTH];
            ptrProg = 0;
            ptrTape = 0;
            loopStack = new Stack<int>();
            jmpTable = new Dictionary<int, int>();

            // start building the jump table:
            // the jump table is a key-value store which matches positions of brackets [ and ]
            // with the positions of their matching counterparts
            // this is also the only place where any error checking is done: all brackets [ and ]
            // must have matching counterparts
            for (int i = 0; i < prog.Length; i++)
            {
                switch (prog[i])
                {
                    case '[':
                        // push its position on the stack
                        loopStack.Push(i);
                        break;
                    case ']':
                        if (loopStack.Count == 0)
                        {
                            // if the stack is empty, then there is no matching counterpart for
                            // this ] bracket
                            // example: program +++[>+++++<-]] would throw this error
                            Console.WriteLine("ERROR: Unmatched ] at position {0}.", i);
                            return;
                        }
                        else
                        {
                            // if the stack is non-empty, then the top element on the stack is
                            // the position of the matching [ bracket
                            // we can now add both positions to the jump table
                            // note: the +1 is because we actually jump to the position right
                            // after the matching counterpart bracket
                            var openPos = loopStack.Pop();
                            jmpTable.Add(openPos, i + 1);
                            jmpTable.Add(i, openPos + 1);
                        }
                        break;
                    default:
                        break;
                }
            }

            if (loopStack.Count > 0)
            {
                // if the stack is non-empty after running over the entire program, then there
                // are one or more [ brackets with no matching counterparts
                // we only show the first one as an error
                // example: program +++[>+++++[>+++++++<-] would throw this error
                Console.WriteLine("ERROR: Unmatched [ at position {0}.", loopStack.Pop());
                return;
            }

            // here we actually start running the program
            while (ptrProg < prog.Length)
            {
                switch (prog[ptrProg])
                {
                    case '>':
                        // increment the data pointer by one
                        // if we have reached the end of the tape, we simply wrap around
                        ptrTape++;
                        if (ptrTape == TAPE_LENGTH)
                            ptrTape = 0;
                        ptrProg++;
                        break;
                    case '<':
                        // decrement the data pointer by one
                        // if we have reached the start of the tape, we simply wrap around
                        ptrTape--;
                        if (ptrTape == -1)
                            ptrTape = TAPE_LENGTH - 1;
                        ptrProg++;
                        break;
                    case '+':
                        // increment the value at the data pointer by one
                        tape[ptrTape]++;
                        ptrProg++;
                        break;
                    case '-':
                        // decrement the value at the data pointer by one
                        tape[ptrTape]--;
                        ptrProg++;
                        break;
                    case '.':
                        // write the value at the data pointer to the console
                        // we need to convert it to a char first
                        Console.Write(Convert.ToChar(tape[ptrTape]));
                        ptrProg++;
                        break;
                    case ',':
                        // read the next character from the standard input stream
                        // in windows, hitting the enter key produces a carriage return and a
                        // line feed (CR+LF), but most brainfuck programs are designed to work
                        // with a line feed only, so we simply ignore the carriage return (13)
                        // also, EOF's are handled by doing nothing
                        var rd = (short)Console.Read();
                        if (rd != 13)
                        {
                            if (rd != -1)
                                tape[ptrTape] = rd;
                            ptrProg++;
                        }
                        break;
                    case '[':
                        // if the value at the data pointer is 0, we jump to the command after
                        // the matching ] bracket
                        // otherwise, we simply increment the program pointer
                        if (tape[ptrTape] == 0)
                            ptrProg = jmpTable[ptrProg];
                        else
                            ptrProg++;
                        break;
                    case ']':
                        // if the value at the data pointer is 0, we simply increment the program
                        // pointer
                        // otherwise, we jump to the command after the matching [ bracket
                        if (tape[ptrTape] == 0)
                            ptrProg++;
                        else
                            ptrProg = jmpTable[ptrProg];
                        break;
                    default:
                        // any other characters not matching a brainfuck token are simply ignored
                        ptrProg++;
                        break;
                }
            }
        }
    }
}
```

Happy brainfucking!
